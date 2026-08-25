# Practical subroutine examples, by pattern family

Worked sketches to draw inspiration from — each names a real recurring shape, its parameter/return
contract, and the design decisions that matter for that shape. None of these are copy-paste SQL; they're
the *contract* (type, return, parameters, atomicity, options) that a real implementation should match,
observed across mature production models. Naming uses the English equivalent of the Dutch conventions
these patterns are commonly found under (`bepaal_`→`calculate_`/`get_`, `bereken_`→`calculate_`,
`controleer_`/`controle_`→`validate_`).

## Calculation and derivation

Centralizes a formula used across screens, reports, and logic — the highest-value, lowest-risk
subroutine family, since it's pure and trivially testable.

- **`calculate_order_total`** — Function, scalar, `return_scalar_dom_id` = a money domain.
  Parameters: `order_id` (input). No side effects, no session dependency beyond the order's own stored
  data. Unit-testable with a single mock order and its lines.
- **`calculate_employee_age`** — Function, scalar, integer/date-diff domain. Parameters: `birth_date`,
  `as_of_date` (both input, both explicit — **never** default `as_of_date` to "today" internally if the
  caller needs reproducible results; pass it in**).
- **`calculate_work_duration_hours`** — Function, scalar, decimal-hours domain. Parameters:
  `start_date_time`, `end_date_time`, `break_minutes` (input). Document rounding and unit explicitly in
  `subroutine_description` — "hours, rounded to 2 decimals" is not inferable from a domain alone.

Design notes: accept every material input as a parameter, never read UI/session state; document
rounding/units/timezone; keep `single_transaction` off (read-only).

## Validation and eligibility

Returns a typed status (and, only if the contract calls for it, a message) rather than overloading the
same return for every kind of failure.

- **`validate_vat_number`** — Function, scalar, boolean domain (or a small status-code domain if there
  are more than two outcomes — "valid," "invalid format," "unknown country," "service unavailable" are
  four distinct outcomes a plain boolean can't express). Parameters: `vat_number`, `country_code`
  (input). If it calls an external validation service, it isn't a pure function anymore — reconsider as
  a Procedure with atomic transaction off, since a network dependency inside a function's implicit
  transaction context is a performance/locking risk (see the SQL style guide's "Subroutines (functions):
  … no cursors, no explicit transactions, no table writes, no messaging — pure functions only" rule; a
  network call belongs in a Procedure, not a Function).
- **`validate_credit_available`** — Function, scalar, boolean or status-code domain. Parameters:
  `customer_id`, `requested_amount` (input). Distinguish "over limit" from "customer not found" from
  "credit check config missing" — three different failure modes a caller needs to branch on
  differently.
- **`validate_email_format`** — Function, scalar, boolean domain. Parameters: `email` (input). Pure
  string validation, no I/O — a good `RETURNS_NULL_ON_NULL_INPUT = Yes` candidate since a null email is
  unambiguously "not a valid email" with no other interpretation.

Design notes: pick a return domain that actually distinguishes the outcomes callers need to branch on;
don't collapse "invalid input," "not found," and "internal error" into the same `false`/`0`.

## Availability and overlap

Common in scheduling, rental, fieldwork, and production planning — and the family most exposed to
concurrency bugs, since two callers can race to reserve the same resource.

- **`get_available_resources`** — Function, table return. Return columns: resource id, start/end of the
  free window, at minimum. Parameters: `resource_type_id`, `window_start`, `window_end` (input).
- **`get_employee_overlap`** — Function, table return (or scalar boolean if the caller only needs a
  yes/no). Return columns (table variant): the conflicting reservation's id and its own start/end.
  Parameters: `employee_id`, `proposed_start`, `proposed_end`, optionally `exclude_reservation_id` (so
  checking an *edit* to an existing reservation doesn't flag itself as its own conflict).
- **`get_vehicle_overlap`** — same shape as above, scoped to a vehicle/equipment resource instead of a
  person.

Design notes: define interval boundaries precisely up front — inclusive vs. exclusive end time,
overnight spans, timezone, canceled/soft-deleted rows, and status filtering (a "tentative" reservation
may or may not count as a conflict, and that's a business decision, not a technical one). **A read-only
overlap check is not sufficient to prevent a race** — back the real reservation with a database
constraint or a locking strategy inside the *booking* procedure, not just this check subroutine; two
callers can both pass the overlap check and then both insert. Write a concurrency test, not just a
serial one — a serial-only test suite reliably misses this exact bug (see the unit-testing section's
concurrency note in the main skill).

## Date and period helpers

- **`get_date_range`** — Function, table return (one row per date/period in the range). Parameters:
  `date_from`, `date_to`, optionally `period_type` (day/week/month). Prefer real date types over
  formatted strings throughout.
- **`spans_midnight`** — Function, scalar, boolean domain. Parameters: `start_date_time`,
  `end_date_time` (input). A small, single-purpose helper — resist the urge to fold it into a larger
  date-utility procedure with a dozen unrelated flags.
- **`get_weekday_name`** — Function, scalar, a translated-text domain if the model supports multiple
  languages; otherwise document the locale/language explicitly in the description rather than baking a
  single hard-coded language into the name.

Design notes: explicitly document timezone, locale, first-day-of-week, fiscal-calendar assumptions, and
whether the result changes with the current date (and thus isn't safely cacheable).

## Integration subroutines

Decompose by **direction, entity, and operation** — e.g. an Exchange-calendar sync family split into
`sync_from_exchange_add_appointment` / `sync_from_exchange_change_appointment` /
`sync_from_exchange_delete_appointment` and the mirrored `sync_to_exchange_*` set, plus separate
retrieval/parsing/checkpoint helpers (`get_exchange_item`, `parse_exchange_attachment`,
`get_exchange_watermark`, `sync_exchange_users`, `check_exchange_server`). This decomposition — one
subroutine per direction × entity × operation, rather than one do-everything sync procedure — is what
keeps each piece independently testable and independently retriable.

Beyond the decomposition itself, an integration subroutine needs:
- A stable external identifier and idempotency — replaying the same sync call twice should not create a
  duplicate.
- Retry classification — distinguish "retry this" (transient network failure) from "don't retry this"
  (the external record was rejected, a business rule failed).
- A watermark/checkpoint (`get_*_watermark`-style helper) so a sync resumes from where it left off
  instead of re-processing everything.
- Timeout handling and structured logging with a correlation id, so a partial failure across a batch is
  traceable to the specific record that failed.
- Explicit ownership of partial failures — if item 47 of 100 fails, does the batch stop, skip, or roll
  back everything? Decide and document it; don't leave it implicit in whatever the loop happens to do.
- Protection of credentials and payloads in any logging this subroutine does.

For a brand-new external integration, weigh a web connection (`thinkwise_software_factory_web_connections`)
or a process-flow connector action before reaching for a database-side integration subroutine — those
are usually easier to operate, monitor, and retry than logic buried inside the database.

## Platform/user helpers

`tsf_user`/`tsf_original_login`-style functions recur across models, often copied from the base model,
and can be configured `INLINE = ON` for SQL Server performance (verified live: `tsf_user` in a real model
carries `with inline = on` in its generated text). Don't casually replace a platform helper — distinguish
*application* identity (`tsf_user()`) from *database connection* identity (`system_user`) before
substituting one for the other; they answer different questions and a naive swap can silently change
audit/security behavior.

## Common failure patterns — watch for these

- **UI logic in a subroutine** — behavior secretly depends on current screen state instead of its own
  parameters.
- **Procedure used as a function** — a command hides mutations behind a value-like name; a caller
  reading `calculate_x` in a `select` has no reason to expect a write.
- **Function with side effects** — query evaluation unexpectedly changes state (the SQL style guide's
  "no table writes" rule for Function-type subroutines exists specifically to prevent this).
- **Generic domains** — everything typed as string/int, weakening the generated contract and API
  metadata.
- **Too many flags** — one routine implementing several loosely related workflows behind a pile of
  boolean parameters; split it instead.
- **Scalar function per row** — fine on one row, disastrous once the same query runs against a large
  table.
- **Manual transaction conflict** — a hand-rolled `BEGIN TRAN`/`COMMIT`/`ROLLBACK` that fights the
  platform's own atomic-transaction handling or a nested caller's transaction.
- **Network call inside a transaction** — locks held for the duration of an unreliable external
  dependency.
- **Null as every failure** — a caller can't tell "not found" from "invalid" from "broken."
- **Output-parameter explosion** — the result is hard to understand or version; move to a scalar/table
  return instead.
- **Hard-coded environment** — paths, credentials, endpoints, company ids, or translated text embedded
  directly in the body instead of read from configuration/domain elements.
- **API checkbox without API design** — flipping `api` on before the external contract (naming,
  versioning, auth, isolation) has actually been designed.
- **Elevated execution without least privilege** — `EXECUTE_AS` or a CLR permission set granting more
  power than the logic actually needs.
- **Old/new copies** — `_old`/`_oud`, `_new`, `_final`, or numbered routines persisting with no
  documented migration owner.
- **No concurrency test** — availability/overlap logic that passes every serial test and races the
  moment two real callers hit it at once.
