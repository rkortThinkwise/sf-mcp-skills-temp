---
name: thinkwise-software-factory-subroutines
description: Reference guide for creating, configuring, calling, and maintaining subroutines (reusable SQL functions/procedures) in a Thinkwise Software Factory model, including role rights and how to call a subroutine from control procedures, views, tasks, and process flows. Use whenever an MCP connector with Software Factory access creates, inspects, or troubleshoots a subroutine or its body SQL.
---

# Creating and Maintaining Subroutines in the Thinkwise Software Factory

Reference for the full subroutine lifecycle: `subroutine` (master object) → `subroutine_parmtr` →
`subroutine_return_col` (table returns only) → `subroutine_option` → control procedure → template →
`template_prog_object_item` → generated function/procedure, deployed to the database, called by other
business logic. The last three steps of that chain are the **general control-procedure mechanics**
already documented in `thinkwise_software_factory_create_control_procedures` — this skill covers what's
specific to subroutines (the contract, the options, choosing type/return shape, role rights, and every
place a subroutine gets *called from*) and hands the SQL-writing/assignment/generation mechanics off to
that sibling skill rather than duplicating them.

Domain key verified live: **`sf/manage_subroutines`** holds `subroutine`, `subroutine_parmtr`,
`subroutine_return_col`, `subroutine_type`, `subroutine_option`, `subroutine_type_option`,
`subroutine_type_option_value`, `subroutine_tag`, `subroutine_parmtr_tag`, and
`role_subroutine_overview.model_rights`. A different domain, `sf/manage_control_procedures`, also
exposes a `subroutine` entity set — that copy is a read-only stub (`model_id`, `branch_id`,
`subroutine_id`, `generated_by_control_proc_id` only, `allow_add/update/delete = false`) meant for
foreign-key lookups from control-procedure entities, **not** the real definition. Always create/edit
subroutines through `sf/manage_subroutines`.

## What a subroutine is, and what it isn't

A subroutine is reusable database business logic with an explicit parameter/return contract, called by
other business logic — never directly by a user-interface component. It's generated as a SQL function,
stored procedure, CLR routine, or (DB2) external routine.

| Need | Best starting concept |
|---|---|
| Reusable database calculation or operation, callable from many places | **Subroutine** |
| User explicitly starts an operation | Task, which may call a subroutine (`thinkwise_software_factory_tasks`) |
| Dynamic UI visibility/editability/mandatory state | Layout logic (`thinkwise_software_factory_create_control_procedures`) |
| Initial values for add/copy | Default logic (same skill) |
| Integrity around insert/update/delete | Handler or database constraint (same skill) |
| A multi-step UI or integration journey | Process flow (`thinkwise_software_factory_process_flows`) |
| A table-shaped subject the UI can browse/screen against | A `tab` with `type_of_table = function` (2) — **not** a subroutine, see below |
| Externally callable business operation | An API-enabled subroutine, or a dedicated web connection endpoint |

**Subroutine vs. `tab`(`type_of_table = function`) — don't conflate these.** Thinkwise has *two*
unrelated ways to model a table-valued SQL function:
- `subroutine` with `return_value = table` — a callable routine with an explicit parameter list,
  invoked from other code (`select * from dbo.get_available_resources(@date)`). No screen, no rows in
  `tab`/`col`.
- A `tab` row with `type_of_table = function` (enum: `table` 0, `view` 1, **`function` 2**, `mqt` 4,
  covered in `thinkwise_software_factory_create_view`) — a table-valued function modeled as a
  screen-viewable *subject*, with real `col` rows, references, screens, and tasks like any other table,
  generated as a function purely so it can take parameters (e.g. a date range) the way a plain view
  cannot.
Pick the `tab`-based route only when the result needs to be browsed/edited on a screen. Pick a
`subroutine` for everything else — it's lighter-weight and has no UI surface to configure.

**Subroutine vs. subflow.** A subroutine is database/business logic with a parameter and return
contract. A subflow is reusable *process-flow* orchestration (process actions and variables) — see
`thinkwise_software_factory_process_flows`.

## Entity map (all in `sf/manage_subroutines` unless noted)

| Entity | Key (adds to parent) | Purpose |
|---|---|---|
| `subroutine` | `subroutine_id` | Master object: type, return shape, atomic-transaction flag, API exposure |
| `subroutine_parmtr` | `subroutine_parmtr_id` | One row per input/output parameter |
| `subroutine_return_col` | `subroutine_return_col_id` | One row per column, **only when `return_value = table`** |
| `subroutine_option` | `subroutine_type_id`, `subroutine_type_option_id` | One row per enabled option (`INLINE`, `EXECUTE_AS`, …) |
| `subroutine_tag` / `subroutine_parmtr_tag` | `tag_id` | Free-form tags, same mechanism as `control_proc`'s dynamic-model tag pattern |
| `subroutine_type` *(read-only)* | `subroutine_type_id` | Platform-provided list of buildable types — **query live, don't hardcode**; varies by `branch_rdbms_type` (a PostgreSQL model only has `Function`/`Procedure`; DB2 adds `External function`/`External procedure`; SQL Server adds `CLR function`/`CLR procedure`/`DLL assembly`) |
| `subroutine_type_option` *(read-only)* | `subroutine_type_id`, `subroutine_type_option_id` | Which options exist per type, and whether each allows a custom value (`allow_custom_value`) |
| `subroutine_type_option_value` *(read-only)* | + `subroutine_type_option_value` | Allowed enumerated values for a non-custom option |
| `role_subroutine_overview.model_rights` | `role_id`, `subroutine_id` | Per-role execute grant — see "Role rights" below |

`subroutine`, `subroutine_parmtr`, and `subroutine_return_col` are all directly writable
(`allow_add/update/delete = true`) — unlike a Task (which needs `task` → `tab_task` → `task_parmtr` in a
strict order, see `thinkwise_software_factory_create_control_procedures`'s "Creating a brand-new Task"
section), a subroutine has **no chicken-and-egg gate**: create the `subroutine` row, then its
`subroutine_parmtr`/`subroutine_return_col` children, in any order, all through ordinary
`stage_resource`/`patch_resource`/`commit_resource` calls.

### `subroutine` fields (verified)

`subroutine_type_id` (must match an existing `subroutine_type` row), `return_value` (byte enum:
`none` 0, `scalar` 1, `table` 2), `return_scalar_dom_id` (domain, only for `scalar`), `return_table_id`
(a free-text label for the returned shape — **not** a foreign key to `tab`; it just names the result set
for documentation/generated-type purposes), `subroutine_description`, `single_transaction` (bool — the
"Atomic transaction" checkbox), `generation_order_no` (int — see "Generation order" below),
`alias_subroutine_id` (subroutine alias), `api` / `basic_api` (bool), `api_alias`, `new_object_status`,
`generated_by_control_proc_id`.

### `subroutine_parmtr` fields (verified)

`dom_id`, `order_no`/`abs_order_no`, `input_parmtr`/`output_parmtr` (bool, independent flags — a
parameter can be neither, though that's unusual), `alias_subroutine_parmtr_id`, `api_alias`,
`subroutine_parmtr_description`, `type_of_subroutine_parmtr_default` (byte enum: `constant_value` 0,
`null_value` 1 — only for Function/Procedure on SQL Server/DB2, per platform docs; **defaults cannot be
functions or expressions**), `default_value`.

### `subroutine_return_col` fields (verified)

`dom_id`, `primary_key` (bool), `mand` (bool), `order_no`/`abs_order_no`. Define every column with a
real domain and correct nullability — a table-valued function should behave relationally.

## Choosing the subroutine type

Query `subroutine_type` for the model first — the buildable set is platform-dependent, not fixed. Then
pick by *behavior*, not availability:

| Type | Use when |
|---|---|
| **Function** | The operation conceptually returns a value with no externally visible state change — calculation, normalization, lookup, or a queryable table/range. Composable in `select`/`join`/`where`/computed columns, but a scalar function evaluated once per row can be expensive at scale — prefer set-based logic or a joinable view for bulk use. |
| **Procedure** | Commands, state changes, integration operations, multi-step work. Harder to compose inside a query; make side effects explicit in the name. |
| **CLR function / CLR procedure** *(SQL Server)* | Only when native SQL genuinely can't implement the requirement and the assembly-deployment/permission-set cost is justified (a specialist library, a legacy integration). New Indicium connectors/process actions are usually easier to operate than database CLR for external integration. |
| **DLL assembly** *(SQL Server)* | The assembly registration backing a CLR function/procedure's `DLL Assembly` option — infrastructure, not business logic itself. |
| **External function / External procedure** *(DB2)* | References an external program — treat as an infrastructure dependency with explicit ownership/deployment/rollback documentation. |

## Choosing the return shape (`return_value`)

- **None (0)** — commands where success/failure and optional output parameters suffice (recalculation,
  sync, cleanup, state transition). Don't hide a useful outcome behind "no return" — a caller often
  needs an inserted id, a row count, or a status; use output parameters for those.
- **Scalar (1)** — exactly one conceptual value: boolean decision, amount, code, date, single id. Pick
  `return_scalar_dom_id` for its real semantics, not just a datatype match — a generic "returns text for
  everything" scalar weakens validation and API metadata.
- **Table (2)** — zero-to-many rows with a stable schema (availability candidates, date ranges,
  validation findings, search results). Define every `subroutine_return_col` with a real domain,
  sequence, nullability, and key where applicable; avoid hidden state changes or row-order assumptions.
- If the request doesn't clearly imply which shape the caller actually needs — e.g. it's ambiguous
  whether a created id/status matters to the caller, or whether a single row vs. many rows is
  expected — ask the user rather than picking from the list above unilaterally
  (`thinkwise_software_factory_mcp_base`'s "Ask, don't default").

## Subroutine options — verified per type (SQL Server model)

Options live on `subroutine_option`, keyed by `(subroutine_id, subroutine_type_id,
subroutine_type_option_id)`. Query `subroutine_type_option` for the target `subroutine_type_id` to get
the live set for the model in hand — the table below is what one SQL Server model exposed, to show the
shape, not a universal list:

| `subroutine_type_id` | `subroutine_type_option_id` | `allow_custom_value` | Values / meaning |
|---|---|---|---|
| Function | `INLINE` | No | `OFF` / `ON` — marks a scalar function inlinable; improves query performance, has usage restrictions |
| Function | `RETURNS_NULL_ON_NULL_INPUT` | No | `No` / `Yes` — skip execution when any input is null; only correct when null propagation is guaranteed for *every* parameter (wrong if null means "use default"/"unbounded"/"not provided") |
| Function | `SCHEMABINDING` | No | `No` / `Yes` — locks referenced-object schema, enables optimizations, constrains future changes to those objects |
| Function, Procedure, CLR function, CLR procedure | `EXECUTE_AS` | Yes (`custom_value` = the account) | Defined privilege boundary only — least-privilege principal, document why elevation is needed |
| CLR function | `RETURNS_NULL_ON_NULL_INPUT` | No | Same semantics as above |
| CLR function, CLR procedure | `DLL Assembly` | Yes (`custom_value` = assembly id) | Which registered `DLL assembly` subroutine backs this CLR routine |
| DLL assembly | `Assembly id` / `DLL file location` | Yes | Free text |
| DLL assembly | `Permission set` | No | `SAFE` / `EXTERNAL_ACCESS` / `UNSAFE` — progressively wider capability and risk; `UNSAFE`/external-access should need explicit architecture/security approval |

To set a non-custom option: add a `subroutine_option` row with `subroutine_type_option_value` set to one
of the values from `subroutine_type_option_value` for that `(subroutine_type_id,
subroutine_type_option_id)`. To set a custom-value option (`EXECUTE_AS`, `DLL Assembly`, `Assembly id`,
`DLL file location`): set `custom_value` instead, leave `subroutine_type_option_value` unset.

DB2 also exposes `DETERMINISTIC` and `MODIFIES SQL DATA` booleans for functions — these must accurately
describe behavior; wrong determinism metadata can lead to invalid optimizer assumptions.

## Plan first

Apply `thinkwise_software_factory_mcp_base`'s "Shared conventions" **Confirm-before-mutate** rule
here: before the first `stage_resource` call, state the proposed contract in plain language and get
the user's explicit confirmation — don't create any row first and adjust after the fact. For a
subroutine, that contract is:

- **Subroutine type** (Function, Procedure, CLR function/procedure, …) — see "Choosing the
  subroutine type" below.
- **Return shape** (`none`/`scalar`/`table`) — and, if scalar, the domain; if table, the column
  list — see "Choosing the return shape" below.
- **Parameters** — name, direction (input/output), and domain for each — see "Parameter design"
  below.
- **`single_transaction`** setting — see "Transaction behavior" below.
- **API exposure** — confirmed off unless the user has actually asked for it — see "Publishing as
  an API" below.

Only start staging `subroutine`/`subroutine_parmtr`/`subroutine_return_col` rows once the user has
confirmed this contract.

## Step-by-step: creating a subroutine

1. **Query `branch_rdbms_type`** (see `thinkwise_software_factory_create_control_procedures`) before
   anything else — it determines both the buildable `subroutine_type` set and the SQL dialect the body
   will be written in.
2. **Create the `subroutine` row**: `subroutine_id` (name for purpose, see "Naming" below),
   `subroutine_type_id` (from the live `subroutine_type` list), `return_value`, and
   `return_scalar_dom_id`/`return_table_id` to match. Leave `api`/`basic_api` off until the contract is
   stable (see "Publishing as an API" below). Decide `single_transaction` now (see "Transaction
   behavior").
3. **Add `subroutine_parmtr` rows**, one per input/output, each with a real business-specific `dom_id`,
   explicit `order_no`, and `input_parmtr`/`output_parmtr` set — don't rely on either flag's default.
4. **If `return_value = table`**, add `subroutine_return_col` rows the same way, plus `primary_key`/
   `mand` where applicable.
5. **Add `subroutine_option` rows** for anything beyond the platform default (see table above).
6. **Grant role execute rights** — see "Role rights" below. Don't skip this; a subroutine with no
   granted role can still generate and deploy cleanly while being uncallable at runtime for every role
   that needs it.
7. **Write and assign the body** — this is where subroutine work rejoins the general control-procedure
   flow. Concretely, verified live:
   - **Code group**: `FUNCTIONS` for a Function returning `none`/`scalar`, `TABLE_VALUED_FUNCTIONS` for
     a Function returning `table`, `PROCEDURES` for a Procedure (any return value). Confirm against
     `code_grp` for the model rather than assuming these three are the only options — CLR/DLL types have
     their own code groups (`CLR_FUNCTIONS`, `CLR_PROCEDURES`, `CLR_ASSEMBLIES`).
   - **`prog_object_id` naming, verified**: `func_<subroutine_id>` for a Function, `proc_<subroutine_id>`
     for a Procedure — this prefix is purely internal bookkeeping.
   - **The real generated database object name is the bare `subroutine_id`**, no prefix — confirmed by
     reading a generated function's text (`create or alter function "tsf_user" (...) ...`). Don't
     expect `func_`/`proc_` to appear in the deployed SQL.
   - **`control_proc_type = 1` (`program_object_item`), `assign_type = 0` (Static)** is the normal shape
     for a subroutine's own hand-written body — one control procedure, one template, one
     `template_prog_object_item` row pointing at `func_<id>`/`proc_<id>`, exactly as documented in
     `thinkwise_software_factory_create_control_procedures`'s "Creating and assigning a control
     procedure — step by step" section. Follow that section verbatim from here, including its dialect
     checklist and the "leave the template blank, generate first to see the real scaffold" step.
   - **Business-logic variables inside the body are just the subroutine's own declared parameters** —
     `@[subroutine_parmtr_id]` in T-SQL (`p_[subroutine_parmtr_id]` in PostgreSQL), not one of the
     Default/Layout/Handler-style variable sets in that skill's `references/code_type_variables.md`.
     There's no separate "input/output variable" table for subroutines because the parameter list *is*
     the variable list.
8. **Generate and verify** — same two-task flow as any other code type
   (`task_generate_code_grp` → `task_add_job_to_generate_object_code`, confirmed via
   `generate_object_code_status`/`generated_code_stale`, then read the generated text). **Known gap for
   a genuinely brand-new standalone subroutine** (verified live in the sibling skill): a subroutine with
   no existing table/view/task to hang off of may never get its placeholder `prog_object` materialized
   through `task_generate_code_grp` — two different attempts both left no row behind, without erroring.
   The `subroutine`/`subroutine_parmtr`/`subroutine_return_col`/`control_proc`/template rows can still
   be fully authored and reviewed through the API regardless. Try the normal generate flow first (it may
   simply work); if no `prog_object` row appears afterward, treat the final generate as a manual step in
   the Software Factory's own UI and say so, rather than continuing to retry alternate API paths.
9. **Copy / rename / delete** — all bound tasks on `subroutine`, verified: `task_copy_subroutine`
   (`from_subroutine_id`, `to_subroutine_id`, `copy_object_assignment` — toggle whether the
   template/functionality assignment comes along), `task_rename_subroutine` (`from_subroutine_id`,
   `to_subroutine_id`), `task_delete_subroutine` (`subroutine_id`). Prefer these over hand-rebuilding a
   near-duplicate subroutine from scratch.

### Generation order (`subroutine.generation_order_no`)

When one subroutine's body calls another, the callee generally needs to exist first at generation time.
This matters more for **functions** than procedures: SQL Server's deferred name resolution lets
`create or alter procedure` succeed even referencing objects that don't exist yet (see the "successful
status is not proof the SQL is valid" note in `thinkwise_software_factory_create_control_procedures`),
but functions don't get the same leniency. Set `generation_order_no` so a subroutine that calls another
subroutine generates *after* the one it depends on, and confirm the dependency chain before assuming a
"Successful" generation status means the call actually resolved.

## Role rights

Every role needs an explicit grant to execute a subroutine — a `prog_object` existing and generating
cleanly says nothing about who can call it at runtime.

- **Where it lives, verified live**: `role_subroutine_overview.model_rights` (queryable directly, though
  it didn't surface through `get_entity_definition` in this session — query it directly rather than
  concluding it doesn't exist). Key: `model_id`, `branch_id`, `role_id`, `subroutine_id`. Fields:
  `granted` (bool — the actual toggle), `role_grp_id`/`role_grp_order_no`/`role_abs_order_no` (grouping/
  ordering for the rights screen), `rights_icon`.
- **One row already exists for every `(role, subroutine)` combination** — verified by reading rows
  across an entire model without filtering by `subroutine_id`. There's nothing to *add*; find the
  existing row for the target role/subroutine and flip `granted` to `true`.
- The `subroutine` entity's bound task `task_from_subroutine_to_subroutine_via_tab_modeler` (labelled
  "Go to Model rights") confirms this is the same screen reachable from the Subroutines modeler in the
  Software Factory UI — use it to cross-check, not as the only path.
- **Not independently verified**: whether a direct `patch_resource` against `granted` on this
  `*_overview` row succeeds, or whether — like `template_prog_object_item` in the control-procedures
  skill — it needs a bound task instead. Try the direct patch first; if it's rejected, look for a bound
  task on the row before concluding the right can't be granted through this API.

## Calling a subroutine — every place it gets invoked from

A subroutine has no UI surface of its own; it only ever runs because something else calls it.

- **From another control procedure (any code type)** — the most common path. Named parameters, always:
  ```sql
  exec dbo.calculate_order_amounts
       @order_id = @order_id,
       @amount   = @amount output;
  ```
  Named calls survive the callee's parameter insertion/reordering; positional calls don't. The caller
  owns the broader transaction unless the subroutine's own contract says otherwise (see "Transaction
  behavior"). `alias_subroutine_id`/`alias_subroutine_parmtr_id`/`api_alias` only affect the *external*
  API surface — an internal caller always uses the real `subroutine_id`/`subroutine_parmtr_id`.
- **From a query, view, or calculated column** — scalar use is concise
  (`select dbo.calculate_age(employee.birth_date, @as_of_date) from employee`), but repeated per-row
  scalar execution can be slow; prefer a set-based expression, join, `apply`, or an inline table-valued
  function for bulk use. A Template-method view (`thinkwise_software_factory_create_view`) can call a
  subroutine directly in its `SELECT`.
- **From a process flow — there is no dedicated "call subroutine" process action, verified.** Two
  mechanisms carry real SQL in a process flow (see `thinkwise_software_factory_process_flows`'s "Process
  logic and control procedures" section):
  1. A `decision` action with `use_processes = true` opts that action into the `PROCESSES` code group;
     its control procedure can call the subroutine directly. Use this for a code-only step between two
     other actions (no task/report/connector attached).
  2. More commonly, an `execute_task`/`execute_system_task` action calls a task whose own **Stored
     procedure**-type logic (in the `TASKS` code group) calls the subroutine.
- **From a task** — the task's Default/Layout/Handler/Task-type control procedure calls it exactly like
  any other control-procedure code type (`thinkwise_software_factory_tasks`).
- **From external applications, via Indicium** — only once `api`/`basic_api` is enabled (see below).
  External clients should never depend on the generated routine's internal name/ordering; call only
  through the published API contract.

### SQL restrictions by code type, when writing a subroutine's own body

From `thinkwise_software_factory_create_control_procedures`'s SQL style guide, the subroutine-specific
rules:
- **Subroutines (functions)**: no cursors, no explicit transactions, no table writes, no messaging —
  pure functions only.
- **Subroutines (procedures)**: avoid cursors where possible; wrap logic in explicit transactions, but
  check `prog_object_item` first — if the code group's own generated wrapper already bookends the
  template with transaction-start/commit items (as Task/Handler code groups do), the hand-written body
  should contain only the business logic, not its own `begin tran`/`commit tran`.

## Transaction behavior (`single_transaction`)

Enable **Atomic transaction** for a procedure when partial completion would violate integrity: creating
a header and lines together, reserving stock and recording the movement, moving a workflow state and
writing required history, applying several linked financial changes. Thinkwise provides nested-
transaction support — when called inside another transaction, final commit belongs to the outer caller;
an error rolls back the earlier statements in the *same* subroutine.

Consider leaving it off when: the operation is read-only, each item is an independent batch unit whose
progress should survive an individual failure, a long-running integration must not hold locks across a
network call, or partial results are explicitly acceptable and recoverable. Non-atomic can perform
better and reduce lock duration, but demands deliberate idempotency and checkpointing in return — never
disable atomicity merely to make a deadlock symptom disappear without finding its real cause (access
order, transaction scope).

When the request doesn't clearly imply whether partial completion would violate integrity — e.g.
it's unclear whether the writes involved are genuinely linked or just happen to run together — ask
the user rather than applying the heuristics above unilaterally
(`thinkwise_software_factory_mcp_base`'s "Ask, don't default").

**Never hand-roll `BEGIN TRAN`/`COMMIT`/`ROLLBACK` that conflicts with the platform's own atomic
handling or a nested caller.** Logging inside a transaction rolls back with the failing work — capture
error data (correlation id, routine, safe-to-log inputs, error number/message, time, caller — never
secrets or full sensitive payloads) into a temp table/variable *before* rollback, then persist it after.

## Error contract

A caller needs a predictable way to distinguish: successful result, expected business rejection,
invalid caller input, authorization failure, not-found/conflict, transient external failure, and
unexpected technical failure. Use a return/output value for expected outcomes the caller is required to
branch on; raise a real error (`tsf_send_message`, never `raiserror` — see the SQL style guide) for
anything that can't complete safely. Don't return `0`, `null`, or an empty table for every kind of
failure — that collapses "not found," "invalid," and "broken" into one indistinguishable outcome.
Preserve the original database error when wrapping it; never swallow an exception after a partial
mutation has already happened.

## Publishing as an API

`api` (Indicium) / `basic_api` (Indicium Basic API) publish the subroutine externally; both are off by
default, and should stay off until the contract is deliberately designed, not merely because an external
consumer wants some data quickly. `alias_subroutine_id`/`api_alias` (on `subroutine`) and
`alias_subroutine_parmtr_id`/`api_alias` (on `subroutine_parmtr`) let the published service/parameter
names differ from the internal Thinkwise convention — useful for a stable external contract, but an
alias is naming only; it creates no semantic versioning.

Checklist before flipping `api` on:
- Stable service and parameter names (via alias if the internal name shouldn't be public).
- Typed required/optional inputs; documented result shape and status behavior.
- Role authorization actually granted for the calling roles (see "Role rights" above) — API access still
  runs through the same per-role execute grant.
- Tenant/row isolation, idempotency for retries, input length/range validation.
- No internal schema or stack-trace leakage in any error path.
- A versioning/deprecation plan — renaming, reordering parameters, changing a domain, or changing a
  table-return column can all break existing clients even though the alias hides the internal name.

## Authorization and execution context

Review, for anything sensitive: role execute rights (above), API publication rights, `EXECUTE_AS`
configuration (least privilege, documented reason for elevation), the underlying tables' own access/
ownership chaining, tenant/company/user filtering, any dynamic SQL, and CLR/DLL permission sets. Never
concatenate untrusted input into dynamic SQL, paths, commands, or URLs — parameterize, and whitelist any
identifier that can't be bound as a parameter. An `EXECUTE_AS` service identity (e.g. for a
`send_email`-style procedure) deserves explicit security review and a test that callers can't turn it
into a general relay.

## Naming and responsibility

Name the *operation*, and let the name reveal whether it calculates, retrieves, validates, or changes
state: `calculate_order_total`, `get_available_resources`, `validate_vat_number`,
`create_invoice_for_order`, `sync_contact_to_exchange`. Dutch-modeled repositories commonly use
`bepaal_` (determine), `bereken_` (calculate), `controleer_`/`controle_` (validate/check) as the
equivalent prefixes — match whichever convention the model already uses, don't introduce a second one.

Avoid `process_data`, `helper`, `do_work`, or version suffixes like `_new`, `_final`, `_old`/`_oud` — a
lingering `_old` routine is a maintenance smell unless it's explicitly kept for a documented migration
reason. A subroutine should have one cohesive responsibility; a procedure that validates, mutates
several domains, sends email, calls an external API, *and* formats a report is difficult to test or
retry safely — split it.

## Parameter design

- Use business-specific domains, not generic strings/numbers — `customer_id` and `employee_id` may both
  be integers while expressing different contracts; don't reuse a domain merely because the underlying
  SQL type matches (see `thinkwise_software_factory_create_control_procedures`'s domain-as-contract
  guidance, which applies identically here).
- Pass stable keys, not display names, for entity identity.
- Avoid dozens of loosely related flags — that's usually a sign the routine needs to be split, or the
  input needs a higher-level shape.
- Keep parameter names consistent with the source column they represent.
- Distinguish "optional" from "unknown" from "intentionally empty," and document units, timezone,
  currency, and format explicitly — nothing in the model enforces this for you.
- Avoid a parameter that secretly depends on a prior call or session state; a platform helper like
  `tsf_user()` is a legitimate context dependency when identity is genuinely part of the rule, but
  document it, and mock it in unit tests where possible.
- Use output parameters for a small number of secondary results (a created id, a status, a message). A
  large output-parameter set is usually better represented as a scalar/table return instead.
- Avoid using one parameter as both input and output unless the mutation is unmistakable from its name —
  it makes both callers and tests harder to read.
- A constant/null default is fine for a stable policy choice or a backward-compatible optional
  parameter; it's risky when it silently hides volatile context (current company, date, language, user)
  that reproducibility actually needs passed explicitly.

## Performance

Common risks: a scalar function evaluated once per row in a large query, row-by-row loops/cursors
instead of set-based logic, repeated lookup queries inside a loop, non-sargable conversions, implicit
conversions from mismatched domains, an unfiltered large table return, a long atomic transaction, a
network call held inside a transaction, excessive logging/payload serialization, CLR startup/deployment
overhead. Test with production-like row counts and parameter distributions; inspect the actual execution
plan, logical reads, duration, and blocking — not just one warm single-row execution. For an expensive
but stable derivation, consider storing the result with explicit invalidation logic instead of
recomputing every call — but never cache a value whose answer depends on security, user, time, or
transaction context.

## Reuse without hidden coupling

A reusable subroutine should depend only on its declared parameters and stable database state. Hidden
dependencies to avoid: the current UI row/variant, undocumented session context, a temp table the caller
happened to create, execution order relative to a previous routine call, a hard-coded company/language/
path/environment value, an ambient transaction assumption, or a lookup by translated text. Avoid a
"do-everything utility" procedure called from dozens of unrelated workflows — high fan-in magnifies the
blast radius of any future change; keep each subroutine's contract narrow and version deliberately when
its semantics must change.

## Versioning and change impact

Treat a subroutine's signature as an internal (or, once `api` is on, external) API. Before changing it,
search for callers and code assignments across the model. Potentially breaking changes: rename/removal,
inserting a parameter into a positional call site, a domain/nullability change, a default-value change,
a return-type or table-column change, new error behavior, a changed transaction boundary, changed
authorization/`EXECUTE_AS`, or a changed side-effect/performance profile. For anything published via
`api`, prefer introducing a new version or an additive optional parameter, migrate consumers, observe
usage, then retire the old contract deliberately — rather than mutating the existing contract in place.

## Unit testing

Subroutines are strong unit-test targets — explicit parameters, explicit return contract. Full mechanics
(the `unit_test_type_id = SUBROUTINE` / `type_of_object = 231` shape, `subroutine_id` as the driving key,
`unit_test_subroutine_parmtr_input`/`_output` for mock parameter values, mock-data wiring for anything
the body reads from a table) are covered in `thinkwise_software_factory_unit_tests` — load that skill
when proposing or building tests for a subroutine rather than re-deriving the entity shape here. At
minimum, cover: the typical result, null inputs, boundary/min-max values, precision/rounding/overflow,
empty and multi-row table results (for table returns), determinism/current-date/current-user
dependencies, complete rollback on a forced failure (atomic procedures), and — for anything guarding a
shared resource (availability/overlap checks) — a concurrency scenario, since serial tests reliably miss
a race that only shows up under real concurrent callers.

## Practical examples and common failure patterns

`references/practical_examples.md` has worked examples across the pattern families that recur in mature
models (calculation/derivation, validation/eligibility, availability/overlap, date/period helpers,
integration subroutines, platform/user helpers), each with a concrete parameter/return sketch — read it
for inspiration before designing a new subroutine from scratch. The same file's closing checklist lists
the failure patterns to watch for (procedure-used-as-function, scalar-per-row-at-scale,
output-parameter-explosion, and the rest).

## Pre-flight checklist

- Query `subroutine_type` (and `branch_rdbms_type`) for the model before assuming a type is buildable —
  the set is platform-dependent.
- Create through `sf/manage_subroutines`, not the read-only `subroutine` stub in
  `sf/manage_control_procedures`.
- `return_table_id` is a free-text label, not a `tab` foreign key — don't go looking for it in the data
  model.
- No creation-order gate like Task's `task`→`tab_task`→`task_parmtr` chain — `subroutine`,
  `subroutine_parmtr`, `subroutine_return_col` are all directly writable in any order.
- Grant role execute rights (`role_subroutine_overview.model_rights`, flip `granted`) — a clean
  generation status proves nothing about who can call the routine.
- Write the body by following `thinkwise_software_factory_create_control_procedures` end to end:
  `FUNCTIONS`/`TABLE_VALUED_FUNCTIONS`/`PROCEDURES` code group, `func_`/`proc_` `prog_object_id` prefix,
  bare `subroutine_id` as the real generated object name, Static assignment,
  `program_object_item`/`control_proc_type = 1`.
- A brand-new *standalone* subroutine's `prog_object` placeholder may not materialize through
  `task_generate_code_grp` — verified gap; be ready to say a manual Software Factory generate pass is
  needed, rather than retrying alternate API paths indefinitely.
- Set `generation_order_no` so a subroutine that calls another subroutine (especially function-calls-
  function) generates after its dependency — functions don't get SQL Server's deferred-name-resolution
  leniency that procedures do.
- There is no dedicated process-action type for calling a subroutine — it's always via a `decision`
  action's `PROCESSES` control procedure or a task's `TASKS` control procedure.
- Internal callers always use the real `subroutine_id`/`subroutine_parmtr_id`; aliases only affect the
  external API surface.
- Keep `api`/`basic_api` off until the external contract is deliberately designed — flip it on last, not
  first.
