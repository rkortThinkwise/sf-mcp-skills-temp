# Choosing a logic concept and designing what it should do

This file is about *design*, not API mechanics: which logic concept a requirement belongs in, what
good and bad logic looks like inside each one, and how to review/test the result. For how to wire a
control procedure through the API once the concept is chosen (code groups, `[PARMTR]` assignment,
generation, enablement gates, business-logic variable names), see `SKILL.md` and
`references/code_type_variables.md`. For the actual SQL body once the type is chosen, see
`references/sql_style_guide.md` and `references/sql_dialects.md`.

## Quick selection guide

| Requirement | Concept |
|---|---|
| Fill or recalculate an entered value | Default |
| Show, hide, lock, or require fields/buttons | Layout |
| Enable tasks, reports, or detail tabs for the selected row | Context |
| Route the next step in a process flow | Process |
| Enforce integrity for all database writes | Trigger/Event, or a declarative constraint |
| Let a user or scheduler execute a command | Task |
| Show a numeric notification | Badge |
| Decide whether auto-refresh is needed | Change detection |
| Replace generated GUI/API insert/update/delete SQL | Handler |
| Reuse a database calculation or command from >1 caller | Subroutine (see `thinkwise_software_factory_subroutines`) |
| Present a reusable relational dataset | View code group (see `thinkwise_software_factory_create_view`) |
| Migrate data during a model-version deployment | Upgrade code group |
| Seed/configure after full deployment | Manual code group |

## Choosing where a rule belongs — decision sequence

Walk this in order; stop at the first "yes":

1. **Can it be modeled declaratively?** — a domain, mandatory property, reference, unique/check/
   foreign-key constraint, permission, filter, or workflow setting. If so, use that instead of any
   control procedure — it's enforced on every write path and the optimizer can reason about it.
2. **Is it only a helpful initial/derived value during entry?** → Default.
3. **Is it only dynamic presentation (visible/editable/mandatory)?** → Layout.
4. **Is it action/detail availability for the selected row?** → Context.
5. **Is it process routing?** → Process.
6. **Is it a user- or schedule-triggered command?** → Task.
7. **Must it protect every database write, regardless of caller?** → constraint or Trigger.
8. **Is custom GUI/API persistence required (multi-table write mapping, special locking)?** →
   Handler, plus real database integrity underneath where needed.
9. **Is it reusable, non-UI logic called from more than one place?** → Subroutine.

**The same business rule can legitimately need two layers, not an either/or pick.** Example: Layout
disables editing on a finalized invoice for usability, while a Trigger/Handler/constraint enforces
the same rule for integrity on every write path (API, import, direct SQL). Don't stop designing once
one layer is in place if the requirement actually has both a UX half and an integrity half.

## Per-concept design guide

### Default
**Good uses:** default current date/user/company; derive a unit after selecting a product; fill an
address after choosing a customer; recalculate a dependent amount; reset values no longer applicable
after a controlling choice changes; move focus to the next logical field.

**Avoid:** integrity rules that must hold outside the UI/API default path (use a constraint/Trigger);
hiding or requiring fields (use Layout); enabling tasks/details (use Context); a large/slow query
after every field exit; overwriting deliberate user input on an unrelated change.

**Best practices:** branch on `cursor_from_col_id` only when behavior genuinely depends on which
field the user changed (see `code_type_variables.md` for the once-vs-every-time pattern); handle a
null `cursor_from_col_id` (add, process-variable, and other non-edit invocations); set every
dependent output predictably, every pass; keep the logic idempotent; use auto-commit sparingly since
it changes transaction/process-flow behavior. Test add, copy, edit, API CRUD, import, task/report
entry, and repeated field changes.

### Layout
**Good uses:** show delivery fields only for physical delivery; make a rejection reason mandatory
after Reject; lock workflow-controlled fields once complete; hide advanced configuration for
irrelevant record types; disable Delete for a finalized record.

**Avoid:** authorization — a hidden or read-only field is not a security boundary (see "Security
guidance" below); changing data values (use Default or task/Handler logic); enforcing database
integrity (use constraints/Trigger/Handler); coloring — use conditional layout configuration instead
(`thinkwise_software_factory_conditional_layouts`).

**Best practices:** Layout is stateless per invocation — start from modeled defaults and output only
purposeful deviations; a field a prior call made mandatory/hidden reverts to the modeled default the
moment a later call doesn't re-state it. Never produce a mandatory-but-hidden state. Keep rules
consistent across Form, Grid, task/report parameters, variants, and roles. Keep it fast — Layout runs
on every row selection and field-exit.

### Context
**Good uses:** enable Approve only for a pending request; hide invoice details when no invoice
exists; show a maintenance detail only for maintainable assets; make Cancel unavailable once
execution has started; activate the most relevant detail tab after a state change.

**Avoid:** field state (Layout); filling data (Default); performing the action itself (Task); using
availability as authorization.

**Best practices:** keep results deterministic from the current row and application context; set
every relevant task/report/detail output every call to avoid stale state carrying over from the
previous selection; avoid expensive existence/count queries on every row selection — use an indexed
flag or predicate instead. Test empty selection, new row, role changes, and parent-detail navigation.

### Process
**Good uses:** choose an approval vs. rejection route; continue only when a connector returned a
valid result; decide whether another step is necessary; initialize/normalize variables for later
actions; route retry/manual-review/stop outcomes.

**Avoid:** replacing the process-flow diagram with one opaque procedure — Process logic should decide
branches, not hide orchestration the flow itself could show; simple unconditional sequencing already
modeled by steps; long-running external calls (use connectors/actions); assuming a task's error
message alone means the action failed — interpret the process-action's actual success state.

**Best practices:** give variables typed domains with explicit input/output mappings; model separate
success, business-rejection, transient-error, and technical-error paths; make retryable actions
idempotent; use the Process Flow Monitor while testing. See
`thinkwise_software_factory_process_flows` for the action/step mechanics.

### Trigger / Event
**Good uses:** maintain required audit/history rows; enforce cross-row integrity a constraint can't
express; keep tightly-coupled derived state in sync; reject an invalid transition from *any* write
path; maintain relationship/link tables.

**Avoid:** pure UI presentation; slow synchronous network calls; large cascades of hidden side
effects; anything already expressible as a foreign key, unique/check constraint, default constraint,
or calculated value — prefer the declarative form.

**Statement-level design:** a trigger fires once per statement, and `inserted`/`deleted` can hold many
rows — never assume a single record; write set-based code (see the cursor-vs-set-based example in
`references/sql_style_guide.md`). Test multi-row insert/update/delete and import/mass-update paths.

**Recursion/ordering/transactions:** document any trigger-to-trigger interaction; avoid updating the
originating table unless recursion is explicitly safe; keep access order consistent to reduce
deadlocks; remember triggers run in the caller's transaction, so raising an error rolls back the
originating action.

### Instead-of trigger
**Purpose:** replaces the normal insert/update/delete action entirely — used for editable views and
specialized child/qualification operations.

**Good uses:** make a multi-table view safely editable; translate one conceptual operation into
several base-table mutations; implement a replacement delete/update with strict business behavior.

**Risks:** the apparent CRUD action may not match the actual mutations performed; generated/UI
expectations around keys, row counts, and refresh can break; multi-row semantics are easy to
overlook; authorization must cover the *real* target tables, not just the view.

**Instead-of trigger vs. Handler:** if the replacement is specifically for GUI/Indicium CRUD and
doesn't need to intercept every database caller, use a Handler instead — it's the narrower, easier to
reason about tool. Reach for an instead-of trigger only when every write path (not just UI/API) must
go through the replacement.

### Task
**Good uses:** approve, plan, allocate, calculate, close, publish, synchronize, generate; operations
needing parameters or confirmation; bulk actions on current/selected/filtered rows; scheduled batch
operations.

**Avoid:** passive recalculation on field change (Default); database integrity that must apply to
every write (Trigger/constraint/Handler); a reusable low-level calculation with no direct user
meaning (Subroutine); one huge task that's really an entire multi-step interactive workflow (Process
flow).

**Best practices:** name with a clear business verb; model parameters with semantic domains and
sensible defaults/layout; ask confirmation for consequential actions (see the confirmation guidance in
`thinkwise_software_factory_build_planner` and message design in
`thinkwise_software_factory_messages`); define current/selected/all-row scope explicitly; choose
await/background execution based on duration and feedback needs; make batch/external operations
idempotent/restartable; use atomic transactions only for work that must succeed as a unit.

### Badge
**Good uses:** open tasks/cases; overdue items; failed integrations; documents awaiting review;
inventory/credit exceptions.

**Avoid:** complex analytics or an exact value that changes every second; a count whose scope differs
from what actually opens when the badge is selected; expensive full-table scans on a short refresh
interval; a sensitive count visible to users who shouldn't see it.

**Best practices:** make the badge query match the target screen's authorization *and* default
filter — a badge showing "12" that opens to an empty/different list is a common, confusing bug;
return a simple numeric value with defined null/error behavior; index the status/owner/due-date
predicates it filters on; pick a refresh interval proportional to business urgency.

### Change detection
**Purpose:** tells Universal UI whether a subject changed immediately before an auto-refresh, to
avoid unnecessary refresh/query work.

**Good uses:** planning/monitoring screens with auto-refresh; expensive subjects whose source changes
infrequently; polling scenarios with a cheap version/timestamp check available.

**Avoid:** running the full expensive subject query just to decide whether to run it — that defeats
the purpose; returning "no change" when authorization/filter-relevant state actually changed; treating
detection as a guaranteed event-delivery mechanism.

**Best practices:** compare a cheap monotonic token — a change version, max mutation timestamp, queue
version, or maintained counter; include every source table that can affect the subject; scope the
token to tenant/user/filter context where relevant. Test inserts, updates, deletes, clock resolution,
and rapid consecutive changes.

### Handler
**Good uses:** CRUD over a complex/updatable conceptual subject; write mapping across several base
tables; special optimistic-locking or persistence behavior; a controlled API/UI write path that
differs from plain database behavior.

**Handler vs. Trigger:**

| Handler | Trigger |
|---|---|
| Invoked for GUI/Indicium CRUD only | Invoked by database DML from every caller |
| Replaces the generated CRUD command | Runs around/instead of the SQL event |
| Uses UI/API input/output context | Uses `inserted`/`deleted` rows |
| Best for application persistence mapping | Best for universal database integrity |

**Do not put an invariant only in a Handler if tasks, imports, integrations, or direct SQL can bypass
it** — a Handler protects one write path, not every write path. Pair it with a constraint/Trigger when
the rule must hold everywhere. See `SKILL.md`'s enablement-flag section
(`use_insert_handlers`/`use_update_handlers`/`use_delete_handlers`) before assuming a Handler runs at
all, and `code_type_variables.md` for the `@upd_[pk_col_id]` vs. `@[pk_col_id]` Update gotcha.

**Best practices:** implement the intended multi-row/key/concurrency semantics fully; return the keys
and refresh results the caller expects; apply authorization/tenant boundaries to the real target
tables; document what bypasses the Handler.

### Subroutine
See `thinkwise_software_factory_subroutines` for the full type/parameter/return/transaction/API/
security/performance/testing guidance — that skill owns this concept end to end. In short: reach for
a subroutine when the logic is reusable across ≥2 call sites, needs to compose inside a SQL query
(function), or needs an external API surface.

## Structural code groups — brief design notes

These generate schema/platform objects rather than per-record business logic; `SKILL.md` covers the
API mechanics for `VIEWS`/`DB`/`UPGRADE`/`MANUAL` already. Design-level notes for the rest:

- **Views** — define and document the row grain up front; avoid accidental many-to-many row
  multiplication from an unguarded join; preserve keys when drill-down/update semantics need them.
  Full guidance: `thinkwise_software_factory_create_view`.
- **Functions/Procedures** (`FUNCTIONS`/`TABLE_VALUED_FUNCTIONS`/`PROCEDURES`) — prefer set-based
  inline table functions over row-by-row scalar functions where the dialect allows it; declare
  null/determinism behavior accurately; keep a procedure's parameter/return/error/transaction
  contract cohesive. See `thinkwise_software_factory_subroutines`.
- **Indexes** — only for a filtered/included-column/specialized index the normal data model can't
  express; validate against an actual query plan and workload, not intuition; check for redundancy
  with modeled/generated indexes first.
- **Constraints/Checks** — the strongest choice for declarative integrity; prefer over Trigger logic
  whenever the rule is a deterministic row/reference constraint. Clean or quarantine invalid existing
  data before enabling one.
- **Sequences** — for database-wide numeric allocation that can't use identity/standard key behavior;
  never promise gapless legal numbering from an ordinary sequence.
- **CLR assemblies/functions/procedures** — only for a capability genuinely unavailable in SQL; review
  assembly trust level (`SAFE`/`EXTERNAL_ACCESS`/`UNSAFE`) and service-account/versioning implications.
  For new external integrations, prefer `thinkwise_software_factory_web_connections` over a CLR
  routine — it's generally easier to secure and operate.
- **Smoke tests** — answer "can the generated object execute/compile in a minimal scenario," not "is
  the business rule correct." They're a deployment-confidence check, not a substitute for the unit
  tests in `thinkwise_software_factory_unit_tests`.

## Performance guidance by concept

Frequently-invoked concepts need extra care about query cost:

| Concept | Fires on |
|---|---|
| Default | Every field exit during entry/import, per row |
| Layout | Every row selection and field change |
| Context | Every row selection and field change |
| Badge | Every configured refresh interval |
| Change detection | Before every auto-refresh poll |
| Trigger | Every DML statement (not per row) |

Practices: use set-based SQL; index the lookup/existence predicates these concepts filter on; avoid a
scalar function call per row; return early on the trigger field/mode where semantically correct; never
call an external network while holding a transaction open; measure with production-like data and
concurrency, not a near-empty dev database.

## Security guidance

- **Layout/Context visibility is not authorization.** A hidden or disabled field/action is a usability
  aid, not a security boundary — anyone with API/direct-SQL access to the underlying table or task
  bypasses it entirely.
- **Default values cannot grant access** — they populate a field the user could already write to;
  they don't widen what the user is allowed to write.
- Tasks, reports, subroutines, process flows, API routes, and the underlying tables all need their
  *own* explicit role rights — a Layout/Context rule hiding them from the UI is not a substitute.
- Trigger/Handler logic must preserve tenant and row boundaries explicitly; don't assume the caller's
  filter already scoped the data correctly.
- Test the direct API/import/task write paths in addition to the Form/Grid path — a rule that only
  fires from the UI is not the same rule as one that fires everywhere.

## Common failure patterns

- **Wrong concept** — data mutation living in Layout, UI state living in Trigger, or an integrity
  rule living only in Default.
- **Presentation mistaken for security** — hidden fields/actions assumed to be protected.
- **Single-row trigger assumption** — a multi-row import or mass update produces wrong results because
  the trigger body assumed one row in `inserted`/`deleted`.
- **Stale outputs** — Layout/Context leaves state from the previous invocation because a call didn't
  re-state every relevant output.
- **Template dependency** — one template silently assumes another already ran or declared a variable
  it relies on (see `SKILL.md`'s guidance on keeping templates atomic and independent).
- **Broad dynamic assignment** — an SQL-assigned control procedure unexpectedly touches far more
  objects than intended; see `SKILL.md`'s dynamic-model-code section.
- **Badge/screen mismatch** — the badge count doesn't match what actually opens (different filter or
  authorization scope) when the user selects it.
- **Metadata-only naming** — `default_table` says where the logic lives, not what it does; see the
  naming guidelines in `SKILL.md`.

## Testing by concept

| Concept | Essential tests |
|---|---|
| Default | Add/copy/update, every controlling field, null cursor, API/import, no overwrite of manual input |
| Layout | Every state combination, roles, variants, mandatory-hidden conflicts, buttons |
| Context | Selection and null selection, actions/details, role, edit changes, no stale output |
| Process | Every branch, variables, success/failure semantics, retry paths |
| Trigger | Insert/update/delete, multi-row DML, rollback, recursion, concurrency, bypass paths |
| Task | Parameters, confirmation, current/selected/all scope, atomicity, background execution |
| Badge | Exact count/scope match to the target screen, authorization, empty/error case, performance |
| Change detection | Every source mutation, rapid changes, transaction visibility, false negatives |
| Handler | Insert/update/delete, keys, concurrency, authorization, both Indicium and UI paths |
| Subroutine | Parameter boundaries, return values, side effects, nulls, API contract |
| View | Row grain, duplicate rows, null joins, authorization |

Use `thinkwise_software_factory_unit_tests` to turn any of these into actual modeled unit tests —
that skill owns the propose → confirm → build workflow and the mock-data mechanics.

## Final review checklist

Before considering a control procedure's *design* (not just its API wiring) done:

- Is the chosen concept the narrowest correct one for this requirement?
- Does its name state business purpose, not code-group metadata (see `SKILL.md`'s naming section)?
- Are all triggering modes and runtime inputs for this concept understood (see
  `code_type_variables.md`)?
- Are all outputs set predictably, every invocation, with no stale state left over?
- Is integrity enforced on every relevant write path, not just the one this concept covers?
- Is authorization handled separately from visibility?
- Is the logic set-based and fast enough for how often this concept fires?
- Are unit/integration/multi-row/concurrency tests proportional to the risk?
