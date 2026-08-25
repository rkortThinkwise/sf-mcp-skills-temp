# Designing Thinkwise unit tests

Grounded in a scan of 5,157 real unit-test definitions across 88 customer models (Task 1,425 ·
Subroutine 826 · Default 737 · Layout 533 · Process 529 · Context 441 · Insert/update/delete handlers
357 · Insert/update/delete triggers 235 · Badge 74). Use this during **phase 1 (propose)** of the
workflow in the main skill file — it's the reasoning behind what to suggest, not the entity mechanics
(see `entity_reference.md` for those).

## What makes a good test

A good unit test demonstrates **one** business rule with controlled input and an explicit expected
result:

- **Arrange**: create only the mock records/inputs the rule actually needs.
- **Act**: execute one default, task, handler, subroutine, or other testable object.
- **Assert**: check the returned values, messages, and/or resulting database state.

*"No database error occurred" is not an assertion.* Prove what changed — or deliberately did not
change. Example shape: *"Given a partially received purchase order, when the 'complete order line'
task is executed, then the line becomes completed and no unrelated lines are modified."*

### Characteristics of a strong test

- Tests one recognizable business rule.
- Has a meaningful name and description (see naming in the main skill file — vague descriptions like
  `"test voor demo"` are the single most common quality gap found across real models).
- Does not rely on data already present in a development database.
- Uses the smallest possible mock dataset — for `DEFAULT`/`LAYOUT`/`CONTEXT`/`BADGE` logic that only
  reads/writes the cursor row's own columns, the smallest dataset is *no `data_set` at all*; input/
  output parameter rows alone are enough. Only reach for mock data when the logic actually queries or
  joins other rows/tables.
- Has an explicit expected output, expected message, or assertion query.
- Tests externally observable behavior, not the SQL implementation.
- Covers both the successful and rejected paths.
- Is deterministic: time, identities, ordering, language, environment don't unexpectedly affect it.
- Can run independently and in any order.
- Leaves no permanent data behind.
- Is linked to its control procedure for traceability and coverage.
- Runs quickly enough to be included in every build.

## Choosing the test type

| Functionality | `unit_test_type_id` | What to verify |
|---|---|---|
| Default procedure | `DEFAULT` | Calculated/defaulted column values, and warnings produced when a value changes |
| Layout procedure | `LAYOUT` | Hidden/visible/mandatory/read-only/editable column states |
| Context procedure | `CONTEXT` | Whether a task, report, detail, or process is available in a particular record state |
| Badge | `BADGE` | Calculated count or other badge value for precisely controlled records |
| Task | `TASK` | Validation, output parameters, messages, database changes caused by the task |
| Process action | `PROCESS` | Process-variable mapping, step results, local process decisions |
| Procedure/function | `SUBROUTINE` | Returned scalar/output parameters, or database state |
| Insert/update/delete handler | `HANDLE_INSERT`/`HANDLE_UPDATE`/`HANDLE_DELETE` | Validation and modifications made before or instead of normal persistence |
| Insert/update/delete trigger | `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL` | Cascading effects and state changes produced after the operation |
| Insert/update/delete statement | (row-filter test, `unit_test_col_filter`) | Which rows change, resulting values, which rows must remain unchanged |

Table-valued functions need an assertion query — their output isn't displayed directly.

## How to decide what tests to create

Start with the **business rule**, not the procedure. For every rule, identify:

1. What condition activates it?
2. What is the smallest input that satisfies that condition?
3. What are the important boundaries?
4. What should be observably different afterward?
5. What must remain unchanged?
6. What should happen for invalid input?

Usually create a separate test for:

- The normal successful path.
- Every materially different business branch.
- Each validation/rejection condition.
- Boundary values immediately below, at, and above a limit (pairs naturally with the `between`/
  `not_between` conditions on `unit_test_col_output`/`_col_filter`).
- `NULL`, empty, zero, and missing-reference cases where meaningful.
- Important state transitions.
- Important database side effects.
- Idempotency, when the operation might run twice.
- Authorization/context conditions that change availability.

You don't need a test for every line of SQL — you need one for every distinct behavior that could
break without another test catching it.

**AND-conditions inside one `IF`**: one scenario per logical group, not per individual condition —
unless the conditions are functionally unrelated (one checks a status, another checks an amount), in
which case each deserves its own sad-flow scenario.

**Filter out scenarios that can't actually be reached.** A Thinkwise procedure is always invoked from
an existing record in the GUI/API context — "record doesn't exist," "primary key is null," "table is
empty" are typically not reachable scenarios for a default/task/handler and don't need a test.

## What to use when

**Output parameters** — direct expected outputs, via the `_output` entities in
`entity_reference.md`. Suitable for defaults' changed column values, layouts' field types, scalar
function returns, task output parameters, process variables, context availability. Usually the
clearest and most maintainable assertion — prefer it over an assertion query whenever the tested object
exposes the result naturally.

**Expected messages** (`unit_test_msg`) — when the business outcome is a warning or error. Real
examples from the scanned models: exceeding the maximum number of packaging layers; executing a task
while a required package is missing; processing an item with one missing dimension; rejecting an
invalid IBAN. Check the *specific business message* — don't treat a raw SQL exception as the expected
behavior when the application should return a controlled validation message. Test both directions: a
sad-path scenario should produce the expected message, and a valid-path scenario should fail if it
unexpectedly produces one. **If a low-level database exception fires before the modeled message is
emitted, the expected message will never be returned** — the test then fails for a reason unrelated to
the rule it's supposed to prove. Validate every prerequisite/mandatory value the scenario needs so
execution actually reaches the intended validation, rather than tripping over an unrelated missing
input first. **Before building the test, confirm the referenced `msg_id` actually exists** (query `msg`
in the messages domain) — control-procedure code can call `tsf_send_message`/`tsf_send_progress` with a
message id that was never created, a real latent bug rather than a test-authoring problem. If it's
missing, flag it to the user rather than silently building the test around a message that will never
resolve correctly; creating the missing `msg` row (and its translation) is a prerequisite fix, not part
of the test itself.

**Assertion queries** (`unit_test.assertion_query`/`unit_test_query`) — when the operation changes
records but doesn't return them, when multiple tables update together, when a trigger creates/removes/
updates related records, when you must confirm nothing changed, when testing a table-valued function,
or when exact row counts/aggregates matter. A good assertion query checks *all* important effects — for
a task that creates an order: exactly one expected order was created, its status/customer are correct,
the expected lines exist, and no duplicate/unrelated rows appeared. Real pattern from the scanned
models, using `tsf_send_assertion_msg`:

```sql
if not exists (
    select 1 from {table}
     where {condition proving the intended change}
)
    exec tsf_send_assertion_msg 0, 'Failed: {what should have happened but did not}';
else if exists (
    select 1 from {table}
     where {condition that must NOT be true — an unrelated row was touched}
)
    exec tsf_send_assertion_msg 0, 'Failed: unrelated row was modified';
else
    exec tsf_send_assertion_msg 1, 'Successful';
```

**Mock datasets** — use whenever the result depends on table/view contents; never depend on live
dev-database data (mock records exist only in the unit test's transaction and roll back afterward). See
"Designing mock datasets" below for sizing, column selection, and reuse guidance — and
`entity_reference.md`'s "Mock data — the API gap" for how mock rows actually get populated through this
connector.

**Preparation queries** — sparingly: set a controlled application setting, fix an effective date/
environment-dependent value, prepare a database object mock rows can't represent, or establish a
precondition that's otherwise impractical. If most of the scenario is hidden in a preparation query,
the test becomes hard to review — prefer visible inputs and mock rows.

**Row filters** (`unit_test_col_filter`, for update/delete statement tests) — test: the intended row
matches; at least one similar row does not match; the correct number of rows changes; an empty match
behaves correctly; a broad/missing filter cannot affect unrelated rows.

## Designing mock datasets

Grounded in a scan of 610 `data_set` definitions across the same models, linked to 2,023 unit-test
attachments. Reuse is common and often healthy — 216 datasets are attached to 2+ tests, 34 to 10+, one
is reused by 80 tests — but the same scan shows the failure mode: datasets that grow into a miniature
production database (one reused set spans 44 tables and 291 selected columns) become a large shared
change surface where an edit made for one test can silently alter dozens of others. Average dataset size
across the scan is 8.4 tables/views and 61.1 selected columns — treat anything far above that as a signal
to split rather than extend.

**The rule that resolves every sizing question below**: mock the business situation, not the database. A
developer should recognize the scenario at a glance — "released order with one unfinished operation,"
"duplicate customer reference" — not have to reconstruct it from a wide grid of columns.

### Dataset anatomy

For any non-trivial rule, reach for up to four kinds of records:

| Record | Purpose |
|---|---|
| Subject | The record the operation acts on. |
| Supporting | Parent/config/reference data the rule actually reads. |
| Contrasting | The opposite state or a different business branch, in the same dataset. |
| Sentinel | An unrelated row that must stay untouched — proves the operation wasn't too broad. |

A task that closes eligible production orders, for example, is better served by four named rows —
`PO_READY` (meets every condition), `PO_OPEN_OPERATION` (blocks on one open step), `PO_ALREADY_CLOSED`
(idempotency), `PO_OTHER_CUSTOMER` (sentinel) — than by four arbitrary orders distinguished only by
numeric id.

### Shape by test purpose

| Purpose | Shape |
|---|---|
| Successful path | Exactly the records required for the rule to fire — nothing extra. |
| Validation (sad path) | The smallest state that violates the rule; assert the message, that state didn't change, and that no side-effect value was written. |
| Duplicate-prevention | An existing row holding the value under test, then pass that same value as input — Thinkwise's own worked example for combining mock rows with matching input. |
| Empty-result | Mock the table but leave it at zero rows, so "nothing qualifies" is tested deliberately rather than by accidentally falling through to whatever live data happens to be present. Use for badges, planning proposals, interface queues, and lookups that can legitimately return nothing. |
| Aggregate/badge | A small, deliberately mixed set — e.g. 3 qualifying + 2 non-qualifying + 1 boundary row — asserting the exact count, not "some number greater than zero." |
| Update/delete | A matching row, a near-miss row, and an unrelated row; drive the row filter (`unit_test_col_filter`) and confirm via assertion query that only the matching row changed. |
| Trigger/cascade | Every record the relationship needs, asserted on both sides: the dependent record's correct creation/update, the parent's correct resulting state, no duplicate dependent record, and an unrelated relationship left alone. |

### Choosing tables and columns

Primary keys are mandatory in a `data_set` (except identity columns), and calculated columns can't be
selected — but beyond those platform rules, select a column only if changing its value could change the
test's outcome:

- Include every column the tested SQL actually reads: filters, joins, status checks, calculation inputs.
- Include columns mandatory to construct a valid mocked record.
- Leave out display-only and audit columns the rule never touches.
- If you can't say which branch a selected column affects, it's probably not needed — the 61.1-column
  average masks datasets built by selecting "everything visible on the tab" rather than what the rule
  reads.

### Constants, expressions, NULL, fallback

Each mock column value can be a constant, a SQL expression, `NULL`, or a fallback to the column's model
default — pick deliberately:

- **Constant** — default choice for anything business-significant (`status = RELEASED`, `blocked =
  false`). Keeps the scenario visible without opening the SQL.
- **Expression** — only when the expression itself is what's required (current Thinkwise user, a fixed
  date literal in the target RDBMS's format, a value no literal can express). Avoid "now"-style
  expressions for anything the assertion later compares against — the same non-determinism risk as an
  uncontrolled current date/user, called out in the "strong test" list above.
- **`NULL`** — deliberate, to express missing-optional-data, required-value validation, or defaulting
  behavior. A `NULL` on a column the record actually requires just produces an unbuildable mock row, not
  a meaningful scenario — don't reach for `NULL` as a way to skip filling in a value.
- **Fallback** — only for structurally-required columns the rule never inspects. Don't fall back a column
  the rule reads: if the model default changes later, the test silently starts covering a different
  scenario without anyone changing the test itself.

### Reuse: when it helps, when it hurts

Good reuse targets genuinely stable reference data — units of measure, status catalogs, a small product
classification, one fixed test user, baseline company config. This is exactly the kind of dataset the
scan shows reused 10, 30, 80 times without issue, because nothing about it is scenario-specific.

Reuse turns risky once a dataset starts describing an entire business flow rather than reference data —
the 44-table/291-column example above. Every test attached to it inherits every future edit to it.
Prefer:

```text
small stable base dataset  +  small scenario-specific dataset
```

over one dataset trying to be both. If only one `data_set` can be linked per test, build the
scenario-specific one and pull in the stable portion via `task_import_unit_test_data_set` (see
`entity_reference.md`) rather than growing a single universal set further.

**Naming** — name the scenario, not the mechanism:

```text
good:  sales_order_partially_delivered, production_order_one_operation_open, ref_basic_units
avoid: general, test_data, dataset_1, everything
```

`data_set_description` should state the scenario and any assumption a reader wouldn't otherwise catch —
same bar as `unit_test_description` in the main skill file's naming section.

### Preparation queries, sparingly

28% of the scanned datasets (172 of 610) use `use_preparation_query` — common enough to be a normal tool,
not exotic, but it should stay the exception per dataset. Reach for it to create a `tsf_user()`-linked
record, alter a view/function for the test's duration, populate something rows can't represent, or fix an
environment dependency. Don't reach for it to insert the scenario's ordinary rows (that's what mock rows
are for), copy unknown live data, delete/truncate real tables, or reimplement the logic under test to
compute the expected result — a preparation query doing the same calculation as the tested code proves
nothing.

### Tables vs. views, and referential integrity

Mocking a view replaces it for the test with a rows-only stand-in (not supported on Oracle); mocking a
table gives it a temporary in-transaction replacement. Mock the view when the code under test *consumes*
it and you want to isolate that code from the view's own joins/external sources — but mock the
*underlying tables* instead when the view's own query is what you're testing.

Thinkwise doesn't require mock rows to satisfy foreign keys, which is what makes small, PK-only datasets
possible in the first place. That's a convenience, not a reason to omit a parent row the rule actually
reads — include a parent/reference record when the code inspects it or its absence would be an
impossible application state; skip it when it only exists to satisfy a constraint the tested logic never
looks at.

### Mocking external integrations

For a web connection, MQTT, ERP/WMS, or file-exchange target, mock the state at the integration boundary
rather than trying to prove the external system is reachable (that's an integration/contract test, see
"Unit test or another test?" below): an inbound-message table holding a fixed payload, an initially-empty
outbound queue, a mapping table with the external identifier, a staging table with both valid and
rejected rows. Assert the normalized records produced, the outbound payload content, the message's
resulting processing state, and that retrying doesn't create a duplicate.

### Smells

- A dataset named `general` (or similar) attached to many unrelated tests.
- Column/table counts far above the 8.4-table/61.1-column scan average with no scenario reason.
- A scenario that only makes sense after reading the preparation query.
- Only positive records — no contrasting or sentinel row.
- An uncontrolled current date/user inside mock data or preparation SQL.
- A dataset that reads like a copy of production rather than a named business situation.

## Examples by functionality

**Defaults** — for a rule deriving a component type: input = component subtype, expected output =
derived type. Add: subtype is `NULL`; whether an existing manually-selected type should/shouldn't be
overwritten; a message test for an invalid combination's warning. Use `unit_test_fixed_parmtr_input`
with `unit_test_parmtr_id = 'cursor_from_col_id'` where behavior depends on which field triggered the
default.

**Layouts** — test the state that makes a field visible+mandatory, a neighboring state that hides it,
and read-only vs. editable separately if they're different rules. Assert the exact field type via
`unit_test_col_type` — don't test whether the UI visually renders a textbox, that's standard runtime
behavior, not a business rule.

**Contexts** — suited to rules like *"the 'Mark complete' task is available only when the order is
confirmed, partially received, or completed, and the line is not already complete."* Cases: every
allowed status if they're different branches, a representative rejected status, already-completed, and
missing/invalid parent data. If the actual concern is whether a user can complete an entire navigation
flow, that's an end-to-end test, not a unit test.

**Tasks and handlers** — test three layers: validation (invalid input → correct message, nothing
changes), main effect (the intended record is updated/created), collateral effect (related records
update correctly, unrelated records stay untouched). Set `should_rollback` where a failed operation
must leave the transaction unchanged. Set `should_abort` only for failures that make subsequent tests
meaningless — not by default.

**Mock data's own transaction rollback is not the same guarantee as `should_rollback`.** Mock data
always runs inside a transaction the platform rolls back at the end of the test, regardless of any
flag — that's cleanup, not proof of anything. `should_rollback` is what actually asserts the *tested
logic itself* reverses its own mutations on error; enabling mock data doesn't substitute for setting
it when the rule under test is "a failed operation leaves no partial change."

**That assertion's own outcome tracking has only been confirmed reliable for `HANDLE_UPDATE`-type
tests.** For `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL`, `TASK`, and `HANDLE_INSERT`/`HANDLE_DELETE`,
the platform has been observed to report "not rolled back" regardless of what the target code actually
did — including a trigger whose generated code contains an explicit rollback statement — which fails an
otherwise-correct test for a reason unrelated to the rule being proven. Until this is re-verified live
against a given connector/model, default `should_rollback` to `false` for those four types and prove
the same intent through `should_abort` plus a message or assertion-query check instead; per the note
above, each run's side effects appear isolated independently of this flag regardless of type, so
turning it off does not reintroduce the cross-test contamination it's meant to prevent.

**Subroutines** — pure scalar functions are ideal: table-driven cases across representative inputs and
boundaries. For table-valued functions, use an assertion query checking row count, required values,
absence of unexpected rows, and ordering only when ordering is actually part of the contract.

**Processes** — unit-test inputs, outputs, variables, and individual decisions (`unit_test_process_
variable_*`, `unit_test_process_step_*`) — don't use a process unit test as a substitute for an entire
user journey. For a process generating/sending an XML document: unit-test data selection and XML
generation separately, assert the generated payload, isolate the external transport; test the real
HTTP/file/broker connection as an integration/contract test; use end-to-end testing for the full
user-triggered workflow.

**Insert/update/delete statements** (row-filter tests, driven by `unit_test_col_filter`) — Thinkwise
auto-adds mandatory input parameters without defaults, excluding identity columns.

- *Insert*: the minimal valid row; defaults/derived values persisted correctly; duplicate/conflict
  rejection; a missing mandatory business relation; audit/history side effects.
- *Update*: one relevant field change; a no-op/unchanged update when the logic detects "nothing
  changed"; a forbidden lifecycle transition; optimistic/concurrency marker behavior where testable;
  a multi-row update (triggers must be set-based — see "Efficient Trigger example" in
  `thinkwise_software_factory_create_control_procedures`'s SQL style guide); history/audit capturing
  both the old *and* new value, not just that a change happened.
- *Delete*: a valid delete; rejection of a referenced/in-use record; soft- vs. physical-delete
  behavior; cascade/cleanup and audit behavior; a multi-row delete.

## Testing time-dependent logic

Time is a frequent source of fragile tests. Prefer passing an effective/reference date into the
subroutine/task under test, or reading a controlled configuration/runtime date mocked for the
duration of the test, over relying on the server's actual current date. Test relative relationships
("30 days after the reference date") rather than hardcoding a year that will eventually expire.

Avoid mixing `getdate()`, `getutcdate()`, local time, and a fixed expected date without documenting
the timezone semantics involved — a test comparing local-time input against a UTC expectation (or vice
versa) can pass or fail depending purely on when and where it runs. Where the rule genuinely depends on
it, test month/year transitions, leap days, daylight-saving boundaries, and inclusive/exclusive cutoffs
explicitly — these are exactly the values a developer is least likely to think to try by hand.

A mock value like `year(getutcdate())` keeps a test perpetually current, but it also obscures the exact
scenario being tested — use it only when the behavior genuinely is relative to the current UTC year,
and make sure the expected output is derived the same way, not hardcoded against today's year.

## Testing user-dependent logic

Logic reading `tsf_user()` or a role/company context needs a controlled identity, not an assumption
about whoever happens to run the test. Don't assume the developer executing the suite has a particular
username, company, or role mapping. Separate the identity lookup from the pure decision where possible
(pass the resolved company/role in as an input rather than re-deriving it from `tsf_user()` inside the
same procedure being tested), and mock the user-to-company/role mapping explicitly rather than relying
on whatever exists in the shared test database. Test the unauthorized and missing-mapping cases too,
not just the happy path — and avoid embedding real personal data (real employee names, real email
addresses) in fixtures built for this.

## Negative and mutation-resistance tests

For any test protecting an important rule, ask whether it would actually fail if the implementation
were subtly wrong: change `>` to `>=`, remove one `where` predicate, invert a status check, return zero
instead of null, update all rows instead of one, skip the rollback, or reuse an old process variable
after a failure. If the test would still pass under any of these mutations, it isn't actually proving
the rule — strengthen the input contrast (add the "near-miss" row from the Dataset anatomy table above)
or tighten the assertion until a plausible defect would be caught.

A mock dataset built with one target row and one near-miss row specifically to catch a missing/loosened
predicate is the same "sentinel" discipline from "Designing mock datasets" above, applied deliberately
as a defect check rather than left to chance.

## Database-platform differences

Run platform-specific variants of a test where the application actually supports multiple database
platforms — don't assume a test written around SQL Server behavior proves the same logic on
Oracle/DB2. Known differences to account for:

- **Oracle's mock fallback behaves differently**: a non-key column left unselected in a mocked table
  can come back `NULL` on Oracle where another platform would apply a fallback/default value — verify
  the actual generated test code (`task_show_unit_test_code`) rather than assuming parity across
  dialects.
- **Oracle doesn't support mocked views** — mock the underlying tables instead when the target platform
  includes Oracle (see "Tables vs. views" above).
- **Oracle disables triggers/constraints during a mocked statement test and cannot reliably re-enable
  them mid-test** — a documented platform limitation. Account for this specifically when designing
  insert/update/delete statement tests on a model that targets Oracle; a trigger-dependent assertion
  that works on SQL Server may not exercise the same path there.
- **Transaction, error, and message behavior varies** across SQL Server, Oracle, and DB2 more broadly —
  verify `should_abort`/`should_rollback` actually produce the expected effect on each targeted
  platform, not just the one used during development.
- **Date, case, collation, decimal, and empty-string/`NULL` semantics can differ** — an assertion that
  quietly depends on one platform's collation or empty-string-vs-`NULL` handling can pass on the
  development platform and fail (or falsely pass) on another.

Use `unit_test_query` (per-`rdbms_type` override of `preparation_query`/`assertion_query`, see
`entity_reference.md`) wherever the generic query text isn't portable across every enabled platform —
the same reasoning as a dialect-specific control-procedure template.

## Unit test or another test?

| Question | Best test |
|---|---|
| Is generated SQL valid and executable? | Smoke test |
| Does one business rule calculate or validate correctly? | Thinkwise unit test (this skill) |
| Does a model validation or naming standard pass? | Model validation |
| Does an API return the promised HTTP contract? | API/integration test (Postman, Insomnia) |
| Does an external ERP/MQTT broker/machine/WMS actually communicate? | Integration/contract test |
| Can a user complete a multi-screen business journey? | Playwright/Testwise end-to-end test |
| Is a changing/exploratory scenario acceptable to users? | Manual test scenario |
| Does it perform adequately under concurrency? | Indicium API load test |

Test customer-specific business logic and interfaces — not standard runtime behavior (navigation,
sorting, filtering, ordinary CRUD). See `entity_reference.md`'s "What was NOT found" for why a
GUI-recorded test-case feature isn't part of this skill's workflow in this connector.

## Recommended team standard

For every new or changed control procedure:

- Describe each business rule in plain language (phase 1 of the main workflow).
- Add a happy-path test.
- Add one test per guard or materially different branch.
- Use mock data for all database dependencies.
- Assert outputs, messages, and database effects explicitly.
- Link the test to the control procedure (`control_proc_id`).
- Run unit tests and smoke tests in CI.
- Add API or end-to-end coverage only when the behavior crosses the unit boundary.
- Treat a deactivated test (`task_deactivate_unit_test`) as temporary technical debt, not a permanent
  state.
- Review coverage by business risk, not only by the percentage shown in a coverage cube. A coverage
  cube distinguishes **Covered** (a program-object item from the control procedure participates
  directly in an active test), **Covered (linked only)** (the test links the control procedure but
  doesn't actually exercise its generated item), and **Not covered** — treat "linked only" as no
  coverage at all when judging real risk; it's a traceability link, not proof the logic runs under
  test.
- Periodically check for `data_set` rows with no `unit_test_data_set` link and remove or repurpose them —
  in a scanned sample, 19 of 610 datasets were unlinked leftovers from renamed/deleted tests.

## Diagnosing a failed test

**First, if the run actually aborted or rolled back (not a clean assertion mismatch), read
`unit_test_result_msg.actual_msg` before guessing** — it holds the raw database error text and usually
points straight at the real cause (a missing deployed stored procedure, a genuine SQL error) rather
than something worth hypothesizing about from the mock dataset or insert order alone; the assertion/
mismatch result entities come back empty in this case since execution never reached the assertion.

Then check in this order before concluding the implementation (or the test) is wrong:

1. **Test design** — type, target object, inputs, expected outputs/messages, and the
   `should_abort`/`should_rollback` flags actually match the intended scenario.
2. **Assertion/preparation query** — read it literally; a query bug produces a failure that looks like
   an implementation bug.
3. **Mock dataset** — tables, selected columns, rows, and values; confirm nothing relevant was left at
   a fallback/null that the tested logic actually reads.
4. **Runtime configuration and deployed generated functionality** — the test database must contain the
   deployed/generated code matching the branch under test; a test run against stale program objects
   produces misleading passes and failures alike.
5. **Generated unit-test code** (`task_show_unit_test_code`) — confirm the test itself compiles to what
   was intended, not just that it "ran."
6. **The linked control procedure and its complete generated program-object code** — read the real
   logic, header/footer included, rather than the template in isolation.

**Don't immediately change the expected output to match the actual result.** First decide, in this
order, whether the *contract* changed (a deliberate behavior change that the test should now reflect)
or the *implementation* changed unintentionally (a real regression the test just caught) — updating the
expected value before that judgment silently launders a regression into "passing."

## Common anti-patterns

- **Execution-only test** — no meaningful expected output or assertion; "no database error" is not a
  result.
- **One giant scenario** — many rules bundled into one test, so any of several failures produces the
  same vague message.
- **Live-data dependency** — the test only passes while a particular row happens to exist in the
  database.
- **Global fixture** — a huge shared dataset makes unrelated tests silently interdependent.
- **Implementation copied into the assertion** — the assertion query reproduces the same logic (and the
  same bug) as the code it's supposed to check.
- **Preparation performs the act** — the preparation query already does what the test claims to prove,
  leaving nothing left to demonstrate.
- **Incidental over-assertion** — asserting values unrelated to the rule, so harmless refactoring breaks
  the test.
- **Missing negative assertion** — the target row changed correctly, but nothing proves every other row
  didn't also change.
- **Null ignored** — an absent output is treated as "nothing to check" instead of the required result.
- **Expected message masks a database error** — the test never actually reaches the intended business
  rule (see "Expected messages" above).
- **Clock-dependent year/date** — a hardcoded date that will eventually expire, or a timezone boundary
  the test doesn't account for.
- **User-dependent result** — the outcome changes depending on which developer's identity runs the
  suite.
- **Single-row trigger test** — a set-based defect (one that only shows up on a multi-row statement)
  goes undetected because the test only ever exercises one row.
- **Abort without state assertion** — the error occurs as expected, but nothing confirms partial
  changes were actually rolled back.
- **External call in a routine unit suite** — a real email/printer/filesystem/web call makes the suite
  slow, flaky, and unsafe to run routinely; isolate the boundary and cover the real call as an
  integration test instead.
- **Coverage-link gaming** — a control procedure is linked to a test without the test actually
  exercising its behavior (see "Covered (linked only)" above).
- **Inactive forever** — a deactivated test accumulates as debt instead of being repaired or removed.
- **Environment mismatch** — model expectations run against stale generated code, producing misleading
  results either way.
- **AI-generated test trusted blindly** — a drafted test's inputs/assertions misunderstand the business
  rule; review it against the actual code the same as any hand-written test before treating it as
  covering what phase 1 proposed.

## Final quality-review checklist

Distinct from `SKILL.md`'s API/mechanics pre-flight checklist — this is about whether the test itself
is actually good, once it's built and running:

- Does the test name state the condition and the expected outcome?
- Does it have exactly one reason to fail?
- Is the unit the smallest practical logic concept for the rule being proven?
- Are all meaningful decision outcomes covered by separate scenarios?
- Are nulls, boundaries, absence, and invalid input covered where they matter to the rule?
- Are inputs controlled and minimal — nothing set that the scenario doesn't need?
- Is the mock data small, coherent, and independent of whatever live data happens to exist?
- Does the test assert exact outputs or post-state, not just "it ran without error"?
- Does it prove important rows did **not** change, not only that the intended row did?
- Are expected messages part of the real contract, not an incidental side effect?
- Is the assertion simpler than, and independent from, the implementation it's checking?
- Is preparation arranging state, not performing the behavior under test?
- Do `should_abort`/`should_rollback` match the intended transaction outcome?
- Are time, current-user, locale, and platform dependencies controlled rather than ambient?
- Are multi-row triggers/handlers tested with more than one row?
- Are external systems isolated here and covered separately by an integration test?
- Is the primary control procedure linked accurately — not just linked for the coverage number?
- Does the test run against current generated/deployed functionality?
- Would a plausible implementation defect actually make this test fail (see "Negative and
  mutation-resistance tests" above)?
- Is the test active, fast, readable, and maintainable?
