# Unit test entity reference

Confirmed live against a connected Software Factory model (domain `manage_unit_tests`, checked
against `manage_control_procedures`/`manage_datamodel`/`manage_cubes` as fallbacks — none of those add
anything beyond what's below). Every entity's key is `model_id, branch_id, ...` — omitted below except
where noted. Re-verify against the live connector before trusting field names across versions, same as
any other skill in this family.

## `unit_test` — the test definition

Key: `model_id, branch_id, unit_test_id`

| Field | Type | Notes |
|---|---|---|
| `unit_test_object_id` | string | |
| `type_of_object` | Int32 enum (742 global values) | Which *kind of object* is under test — Table, Task, Process flow, Subroutine, Report, … Picks which scalar key field below is populated. Observed: `0`=Table, `11`=Task, `30`=Process flow, `231`=Subroutine. Confirm others via `unit_test_type_object` rather than guessing. **The same field name reappears on `unit_test_input_parmtr_overview`/`unit_test_output_parmtr_overview` with an unrelated value domain** — there it means *kind of parameter*, not kind of object (confirmed live: `1`=col, `32`=task_parmtr, `826`=the fixed-parameter catalog entry, e.g. for `cursor_from_col_id`). Don't assume the two columns share meaning just because they share a name. |
| `unit_test_type_id` | string | Which *kind of test* — `DEFAULT`, `LAYOUT`, `CONTEXT`, `BADGE`, `TASK`, `PROCESS`, `SUBROUTINE`, `HANDLE_INSERT`/`HANDLE_UPDATE`/`HANDLE_DELETE`, `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL` observed live. Confirm the full catalog via `unit_test_type`/`unit_test_type_object` — don't assume this list is exhaustive. |
| `unit_test_description` | string, translatable | Full sentence: condition + expected result. Never leave vague/empty — see main skill file. |
| `should_abort` | bool | Stop the whole automated run if this test fails. Reserve for failures that make subsequent tests meaningless — not a default. |
| `should_rollback` | bool | Asserts that the tested logic itself reverses its own mutations. **Its outcome tracking (`was_rolled_back` on the result) has only been confirmed reliable for `HANDLE_UPDATE`-type tests.** For `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL`, `TASK`, and `HANDLE_INSERT`/`HANDLE_DELETE`, `was_rolled_back` has been observed to stay `false` regardless of the target code's actual behavior — including cases where the code contains an explicit rollback statement — which spuriously fails an otherwise-correct test. Default this to `false` for those four types and rely on `should_abort` plus the message/assertion checks instead; repeated re-runs against the same hardcoded fixture ids did not produce duplicate-key errors, indicating each run's data changes are isolated independently of this flag. See "Tasks and handlers" in `test_design_guide.md` for the fuller reasoning. |
| `use_preparation_query` | bool | Gate for `preparation_query`. **Toggling this (or `use_assertion_query` below) can reset the *other* query-text field to null/placeholder as a side effect of the record's own dynamic layout** — confirmed live, reproduced repeatedly. After any patch that touches either flag, re-read `preparation_query`/`assertion_query` and re-patch if needed rather than assuming a value set earlier in the same edit session survived. |
| `preparation_query` | string | SQL run before the test — set a controlled setting, fix an effective date, prep something mock rows can't represent. Keep sparing; if most of the scenario lives here, the test is hard to review. **If this inserts an explicit value into an identity-typed primary key** (the usual advice — see "Always mock identity/PK values explicitly" below, which applies here too even when bypassing the `data_set`/`tab_data` pipeline), wrap that table's insert in `SET IDENTITY_INSERT <table> ON` / `OFF` (SQL Server; confirm the equivalent on other platforms via `branch_rdbms_type`) or the insert is rejected outright with an identity-column error. |
| `use_assertion_query` | bool | Gate for `assertion_query`. See the toggle-reset caution on `use_preparation_query` above — it applies symmetrically. |
| `mock_data_used` | bool | Whether a linked `data_set` is actually applied. |
| `assertion_query` | string | SQL asserting the observable outcome — row counts, exact values, absence of unexpected rows. Call `tsf_send_assertion_msg` (see examples below) rather than relying on a bare query result. |
| `tab_id` | FK, nullable | Populated for table-scoped types (`DEFAULT`, `LAYOUT`, `CONTEXT`, `BADGE`, `HANDLE_*`, `TRIGGER_*`). |
| `task_id` | FK, nullable | Populated for `TASK`. |
| `report_id` | FK, nullable | Populated for a report-context test. |
| `process_flow_id` | FK, nullable | Populated for `PROCESS`. |
| `process_action_id` | FK, nullable | Populated for a process-action-scoped `PROCESS` test. |
| `subroutine_id` | FK, nullable | Populated for `SUBROUTINE`. |
| `control_proc_id` | FK | The control procedure whose logic is under test — traceability link; **not always the same value as `task_id`**, verify separately. |
| `result_status` | Byte enum | Last-run outcome: `successful=0, model_mismatch=1, failed=2, not_executed=4, executing=5, scheduled=6`. Distinct from the `test_unit_test` job-status enum below — don't conflate. |
| `start_date_time` | datetime | Last run start. |
| `active` | bool | Inactive tests don't run in a batch — prefer deactivating over deleting a temporarily-broken test. |
| `insert_user`/`insert_date_time`/`update_user`/`update_date_time` | audit | Standard audit columns — present, so this is a normal writable entity. |
| `generated_by_control_proc_id` | FK, nullable | Set only if a dynamic-model procedure owns/regenerates this row. |

**No `$expand` from `unit_test` to any `unit_test_col_*`/`*_parmtr_*`/`unit_test_process_*` child** —
they relate only by shared key. Query them directly, filtered on `unit_test_id`. The only true
navigation properties confirmed on `unit_test` are: `unit_test_col_filter` (as `.trigger`),
`unit_test_data_set` (as `.mock_data`), `unit_test_input_parmtr_overview`,
`unit_test_msg`, `unit_test_output_parmtr_overview` (+ `.condition`/`.parmtr_type` variants),
`unit_test_progress`, `unit_test_query`, `unit_test_result`.

**`unit_test.model_id`/`branch_id` are hidden and reject direct writes on a fresh insert** — a
top-level add leaves them null with no way to patch them (a lookup field like `tab_id`/`control_proc_id`
then 403s because the model/branch context is missing). Adding the row as a dependent record under
`control_proc` does resolve that hidden context and pre-fill `control_proc_id` — **but `type_of_object`
itself has been confirmed to still reject a patch as an unknown property on that path**, even though the
staged record's dynamic field list shows it as editable; a field appearing editable in that per-instance
response is not proof the same access path's static schema actually accepts a write to it.

**The reliable way to create a new `unit_test` row, confirmed live across `DEFAULT`, `HANDLE_UPDATE`,
`TASK`, `TRIGGER_UPD`, `HANDLE_INSERT`, and `HANDLE_DELETE`: use `task_copy_unit_test` (see the task
table below) to copy *any* existing `unit_test` row — regardless of its type — as a scaffold, then
override `to_unit_test_id`/`unit_test_type_id`/`type_of_object`/the target field
(`tab_id`/`task_id`/etc.)/`control_proc_id` on the same staged task call before committing.** This
resolves the hidden `model_id`/`branch_id` context, makes `type_of_object` and the target field settable
together, and avoids the dependent-add path's `type_of_object` rejection entirely. Only fall back to a
plain dependent-add under `control_proc` for fields it disallows changing (rare) — don't treat it as the
default creation path. After the copy, re-check for copied-over child rows before adding new ones (see
the `task_copy_unit_test` entry below) — a column name shared between the source and target table for
unrelated reasons can carry a stray input row across even when the object types differ.

## `unit_test_condition` — 16 values, confirmed live

A naive guess based on common comparison operators produces an 8-value enum missing half the real
values. The real enum:

```
equal_to = 0
not_equal_to = 1
greater_than = 2
smaller_than = 3                    -- NOT "less_than"
greater_than_or_equal_to = 4
smaller_than_or_equal_to = 5        -- NOT "less_than_or_equal_to"
between = 6
starts_with = 7
contains = 8
does_not_contain = 9
is_empty = 10                       -- NOT "is_null"
is_not_empty = 11                   -- NOT "is_not_null"
does_not_start_with = 12
not_between = 13
ends_with = 14
does_not_end_with = 15
```

`between`/`not_between` pair with the sibling `expected_output_value_until`/`filter_value_until`
column present on every output/filter entity below, for range checks (a boundary "immediately below,
at, and above a limit" scenario is a natural `between`/`not_between` pair).

**Writing this field on an output/filter row needs the same numeric-string-plus-explicit-data-kind
treatment as `type_of_data_value` on `tab_data_col_overview`** (see `mock_data_guide.md`) — patching it
with the enum's display label (e.g. `"equal_to"`) 400s with an invalid-input error; patch the numeric
string (`"0"`, `"1"`, …) with the value explicitly marked as a data value instead.

## `unit_test_parmtr_type` — 2 values

`parmtr_type` ("type") / `parmtr_mand` ("mand") — used on `unit_test_col_type`,
`unit_test_task_parmtr_type`, `unit_test_ref_output`, `unit_test_tab_report_output`,
`unit_test_tab_task_output` to select whether the expected-output row is checking the field's *type*
(hidden/visible/read-only/editable, or task/report/detail availability) or its *mandatory* flag.

## Input/output/filter/handler child entities

All keyed `model_id, branch_id, unit_test_id, ...` plus the columns below. None have a navigation
property from `unit_test` — filter directly on `unit_test_id`.

| Entity | Extra key columns | Value columns | Use for |
|---|---|---|---|
| `unit_test_col_input` | `tab_id, col_id` | `input_value` | Default test **and** insert/update/delete handler test: input column value. **Confirmed live: `HANDLE_INSERT`/`HANDLE_UPDATE`/`HANDLE_DELETE` tests write their input columns through this same entity, not a separate `unit_test_col_handler`** — that entity exists and is queryable but was never where a handler test's actual input rows landed in practice. Don't build a handler test's input path around `unit_test_col_handler`. **If the handler's own custom code reads the framework's row-identifying `@upd_<col>` parameter directly** (read the generated code first to check) **rather than only the plain column parameter, add that as its own extra input row too, with `parmtr_id` set to `upd_<col_id>`** — every generated test script otherwise leaves `@upd_<col>` null, which silently no-ops any handler branch that depends on it, even though the row's plain-column input was set correctly. |
| `unit_test_col_output` | `tab_id, col_id` | `unit_test_condition`, `expected_output_value`, `expected_output_value_until` | Default test: expected calculated/defaulted value. |
| `unit_test_col_type` | `tab_id, col_id, unit_test_parmtr_type` (condition is part of the key) | `expected_output_value`, `expected_output_value_until` | Layout test: expected field type or mandatory state. |
| `unit_test_col_filter` | `tab_id, col_id, unit_test_condition` (condition is part of the key) | `filter_value`, `filter_value_until` | Row-filter test (update/delete statement): row-matching condition. **Also the mechanism for targeting a specific existing row on a `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL` test** — setting a primary-key column as an ordinary input value on those types is rejected outright (403), so identify the row via a filter instead (typically `col_id` = the PK column, `unit_test_condition` = `equal_to`, `filter_value` = the row's id). **Confirmed directly addable at the top level** (with explicit `model_id`/`branch_id`/`unit_test_id` in the same call) — unlike the input/output overview entities below, it is *not* reachable via a dependent add under `unit_test` (no matching detail navigation was found from that parent). |
| `unit_test_col_handler` | `tab_id, col_id` | `input_value` | Exists and is queryable, but confirmed live to be a dead end for handler-type test authoring — see `unit_test_col_input` above for where these values actually go. |
| `unit_test_task_parmtr_input` | `task_id, task_parmtr_id` | `input_value` | Task test: input parameter value. |
| `unit_test_task_parmtr_output` | `task_id, task_parmtr_id` | `unit_test_condition`, `expected_output_value(_until)` | Task test: expected output parameter value. |
| `unit_test_task_parmtr_type` | `task_id, task_parmtr_id, unit_test_parmtr_type` | `expected_output_value(_until)` | Task test: expected parameter type/mandatory state. |
| `unit_test_subroutine_parmtr_input` | `subroutine_id, subroutine_parmtr_id` | `input_value` | Subroutine/function test: input parameter value. |
| `unit_test_subroutine_parmtr_output` | `subroutine_id, subroutine_parmtr_id` | `unit_test_condition`, `expected_output_value(_until)` | Subroutine/function test: expected return/output value. |
| `unit_test_process_variable_input` | `process_flow_id, process_variable_id` | `input_value` | Process test: input process variable value. |
| `unit_test_process_variable_output` | `process_flow_id, process_variable_id` | `unit_test_condition`, `expected_output_value(_until)` | Process test: expected output process variable value. |
| `unit_test_process_step_input` | `process_flow_id, process_step_id` | `input_value` | Process test: input at a specific step. |
| `unit_test_process_step_output` | `process_flow_id, process_step_id` | `unit_test_condition`, `expected_output_value(_until)` | Process test: expected value/decision at a specific step. |
| `unit_test_ref_output` | `ref_id` | `unit_test_parmtr_type` (scalar, not key), `expected_output_value(_until)` | Context test: expected reference/detail-tab availability. **Output-only, no `_input` counterpart.** |
| `unit_test_tab_report_output` | `tab_id, report_id` | `unit_test_parmtr_type`, `expected_output_value(_until)` | Context test: expected report availability. **Output-only.** |
| `unit_test_tab_task_output` | `tab_id, task_id` | `unit_test_parmtr_type`, `expected_output_value(_until)` | Context test: expected task availability. **Output-only.** |
| `unit_test_fixed_parmtr_input` | `unit_test_parmtr_id` | `input_value` | Shared fixed parameters — chiefly `cursor_from_col_id` (which column triggered a default). |
| `unit_test_fixed_parmtr_output` | `unit_test_parmtr_id` | `unit_test_condition`, `expected_output_value(_until)` | Expected value of a fixed parameter. |
| `unit_test_msg` | `expected_msg_id` | — (nav `lookup_expected_msg_id` → `msg`) | Expected validation/warning message for the sad-flow path. |
| `unit_test_query` | `unit_test_id, rdbms_type` (enum: `sqlserver=0, iseries=1, oracle=3, postgresql=4`) | `preparation_query`, `assertion_query` | Per-dialect override of the generic prep/assertion query on a multi-RDBMS model — same reasoning as `control_proc_template` per dialect (see `thinkwise_software_factory_create_control_procedures`). |

**The table above is correct for reads (filter directly on `unit_test_id`) but a direct `add` on most
of these entities is rejected outright** — verified live on `unit_test_col_input`, and the same shape
of rejection as `template_prog_object_item` in the control-procedures skill. **`unit_test_col_filter`
is the confirmed exception — it's directly addable at the top level** (see its row above); don't route
row-filter writes through the overview entities below, they're for input/output values, not filters.
**The real way to write an input or output row is `unit_test_input_parmtr_overview` / `unit_test_output_parmtr_overview`**, added
as a dependent record under `unit_test` (only one detail navigation candidate, so no ambiguity there).
Set `parmtr_id` to the column id, task-parameter id, subroutine-parameter id, *or* fixed-parameter id
(e.g. `cursor_from_col_id`) — the row auto-resolves the matching `unit_test_parmtr_id` and the
parameter-kind `type_of_object` (see the note on that field name above) regardless of which kind of
parameter it turned out to be. Then set `input_value`, or `unit_test_condition` +
`expected_output_value` for an output row. **To leave a column/parameter at its natural blank/null
state, don't create a row for it at all** — `input_value`/`expected_output_value` are mandatory on a
row that exists, so a null-input scenario is expressed by omission, not by an empty value (which fails
to commit). **The same routing applies to delete.** Removing a stray row — e.g. one that carried over
incorrectly from a cross-table `task_copy_unit_test` because the source and target tables happened to
share a column/parameter name — must go through the same overview entity; a direct delete on
`unit_test_col_input`/`unit_test_col_output` themselves is rejected the same way a direct add is.

**Datetime `equal_to` comparisons are a literal string match, not a typed compare.** Entering an
ISO or locale-formatted datetime as `expected_output_value` gets silently reformatted for display, but
the runtime assertion compares that stored string byte-for-byte against the actual output string — a
logically identical instant in a different format (e.g. `01/01/2026 10:00:00` vs.
`2026-01-01 10:00:00.0000000`) fails the test even though the business logic is correct. If a `DEFAULT`/
`TASK`/etc. test with an `equal_to` datetime output fails despite the logic looking right, check
`unit_test_result_parmtr.output_value` for that run (see "Object catalog and results" below) and
re-enter `expected_output_value` in that exact raw format (SQL Server: `yyyy-MM-dd HH:mm:ss.fffffff`,
7 fractional digits) rather than guessing at a "nicer" format.

**The same string-match behavior applies to money/decimal domains.** A domain stored with fixed
decimal places returns its full formatted string (e.g. `"0.00"`), so an `expected_output_value` of
`"0"` fails an `equal_to` check even though the value is numerically identical — match the actual
formatted precision, not the shortest numeric representation.

## Mock data — `data_set` and the API gap

**Not every test needs one.** A `data_set` link is only necessary when the control procedure's logic
queries or joins rows/tables beyond its own input/output parameters. A `DEFAULT`/`LAYOUT`/`CONTEXT`/
`BADGE` test whose logic only reads and writes the cursor row's own columns — the common case for a
simple default expression — needs no `data_set` at all; the input/output parameter rows above are
sufficient. Only reach for the steps below when the scenario genuinely depends on other rows existing.

| Entity | Key | Notes |
|---|---|---|
| `data_set` | `data_set_id` | `data_set_description`, `use_preparation_query`, `preparation_query`, audit columns. Nav props: `transl_model_id`/`lookup_model_id`/`list_model_id` only — **no nav chain to child row data**. |
| `unit_test_data_set` | `unit_test_id, data_set_id` | Junction linking a test to a data set (nav'd from `unit_test` as `.mock_data`). Bound task `task_from_unit_test_data_set_to_data_set_via_unit_test_modeler` opens the Software Factory "Mock data" screen for this link. **Adding this as a dependent record under either `unit_test` or `data_set` fails outright** (a detail-navigation-not-found error) **even with full permissions** — the relationship is exposed under a qualified/compound target name the connector's own parent-detection can't resolve; don't keep retrying that route. **A fresh top-level add is a different story and is permission-gated like everything else in this file, not a separate structural gap**: stage an empty add first — if `model_id`/`branch_id` come back `hidden` in the staged fields, that's the standard missing-grant signal (see "Permissions" in `mock_data_guide.md`). Once they come back `editable`, patch all four key fields (`model_id`, `branch_id`, `unit_test_id`, `data_set_id`) together in one call — confirmed live to commit cleanly. Patching only `unit_test_id`/`data_set_id` while `model_id`/`branch_id` are still unset 403s regardless of which property is patched first, which can look like a deeper bug but is just the same missing-context symptom under a misleading error. If the grant genuinely can't be obtained, fall back to `preparation_query`/`assertion_query` (see `mock_data_guide.md`'s note near the top) instead of the `data_set`/`tab_data` pipeline for that test. |

**`data_set_tab`, `data_set_col_selection`, `tab_data`, `tab_data_col_overview` — the actual row/column
mock-data entities — DO exist as queryable and writable OData entity sets, confirmed live end-to-end
(create table → select columns → create row → set values, including FK-chained multi-table mocks).**
An earlier version of this doc claimed these don't exist as entity sets at all — that was wrong; they
were reachable all along, just gated by table-level permissions on `data_set_tab`/`tab_data` that
weren't granted on the connector/user checked at the time. **See
`references/mock_data_guide.md` for the full, confirmed-working recipe** — do not fall back to a manual
Studio step without first checking whether this is actually a permissions gap (missing `allow_add`/
`task_create_tab_data` in `get_entity_definition`, not a missing entity set).

Two API paths remain useful even with full permissions:
- **`task_import_unit_test_data_set`** (bound to `unit_test_data_set`; param `from_unit_test_id`) —
  clone an existing data set with a matching shape, then adjust the per-test input/output values. Still
  the fastest option when a close-enough donor already exists.
- **A manual Software Factory step** through the Mock data screen remains the true last resort only if
  permissions genuinely can't be granted, or there's no existing `data_set` row anywhere in the model to
  clone the very first one from.

`sample_data_set` (`manage_datamodel`, key `sample_data_set_id` Int32, column `data_set_name`) is an
**unrelated** feature — domain-level example values, not unit-test mock data. Don't confuse the two
despite the similar name.

## Object catalog and results (read-mostly)

| Entity | Purpose |
|---|---|
| `unit_test_object`, `unit_test_object_control_proc`, `unit_test_object_parmtr` | Which object types/control-procs/parameters each `unit_test_type_id` supports — the schema behind the "Choosing the test type" table. **Also useful as a discovery tool**: query `unit_test_object` filtered by `unit_test_type_id` to list every pickable target (table/task/etc.) of that kind on the branch, keyed by its real `tree_unit_test_object_id`/`unit_test_object_id` — the fastest way to confirm the actual `tab_id`/`task_id` bound to a `control_proc_id` when the task or table name doesn't obviously match the control procedure's name. |
| `unit_test_type`, `unit_test_type_object`, `unit_test_type_fixed_parmtr` | Catalog of test types and which object-type/fixed-parameter combos each supports — the authoritative source for `type_of_object` values and the full `unit_test_type_id` list, instead of hardcoding either. |
| `unit_test_result`, `unit_test_result_assertion`, `unit_test_result_mismatch`, `unit_test_result_msg`, `unit_test_result_parmtr`, `unit_test_result_parmtr_type`, `unit_test_result_parmtr_overview` | Per-run result detail (key includes `unit_test_result_id` Int64) — what actually happened on the last execution, beyond the summary `unit_test.result_status`. **For a crashed run (the transaction was aborted/rolled back, not a clean assertion mismatch), check `unit_test_result_msg.actual_msg` first** — it holds the actual raw database error text (e.g. a missing-stored-procedure error), not just expected-message assertions for the sad-flow path; `unit_test_result_assertion`/`unit_test_result_mismatch` come back empty in this case since execution never reached the assertion. |
| `unit_test_progress` | Live progress row (key `model_id, branch_id`) for a currently-running batch. |
| `test_unit_test` / `test_unit_test_overview` | The batch-execution job record (key `job_id` Int64). Status enum: `scheduled=0, executing=1, wait_for_user=2, successful=3, failed=4, cancelled=5, aborted=6, warning=7, info=8`. **Different enum from `unit_test.result_status`** — this one tracks the batch job, that one tracks the individual test's last outcome. |
| `quality_dashboard_unit_test_coverage`, `quality_dashboard_unit_test_success`, `unit_test_control_proc_coverage_cube(_modeler)`, `unit_test_prog_object_coverage_cube(_modeler)`, `unit_test_run_cube(_modeler)` | Coverage/success dashboards and cubes — reporting, not authoring. |

## Tasks

All confirmed live via `get_task_definition` in `manage_unit_tests`.

| Task | Bound to | Parameters | Purpose |
|---|---|---|---|
| `task_add_job_to_test_unit_test_single` | `unit_test`, `test_unit_test_overview` | optional `host`, `db_name` (+ `unit_test_id` on the overview variant) | Execute one selected unit test. |
| `task_add_job_to_test_unit_test` | `unit_test`, `test_unit_test_overview` | optional `host`, `db_name` | Execute the whole active suite. |
| `task_cancel_job` | `test_unit_test`, `test_unit_test_overview` | (key `job_id`) | Cancel a running batch. |
| `task_show_unit_test_code` | `unit_test` | mandatory `rdbms_type`; optional output `sql` | Preview generated test SQL without running it. |
| `task_enrichment_generate_unit_test` | `unit_test` | mandatory `description`, `unit_test_type_id`, `type_of_object`; optional `tab_id`/`task_id`/`report_id`/`process_flow_id`/`process_action_id`/`subroutine_id`/`control_proc_id` | AI-drafts a test from a natural-language description — a drafting aid for phase 3, review before trusting it covers the approved phase-1 scenario. |
| `task_copy_unit_test` | `unit_test` | mandatory `from_unit_test_id`, `to_unit_test_id`, `unit_test_type_id`, `type_of_object`; optional target-object fields | Clone a test definition to a new id (definition only, not the mock-data link). **This is also the reliable primary path for *creating* any new `unit_test` row** — see the note on `unit_test.model_id`/`branch_id` above; copy from any existing test regardless of its type and override every field in the same call. Copied input/output rows whose column doesn't exist on the new target are dropped automatically — **except when the source and target tables happen to share a column name for unrelated reasons, in which case that row incorrectly carries over.** Always list the new test's input/output rows after a cross-table copy and delete anything that doesn't belong to the actual scenario. **If the source's `type_of_object` already matches the target (e.g. copying between two `TASK`-type tests), omit it from the patch** — it comes back read-only in that case, and including it fails as not-editable. **A combined patch that includes both `to_unit_test_id` and `unit_test_type_id` can silently clear `to_unit_test_id` as a layout side-effect**, even though it was applied first in the same call — re-read the returned field values after every patch and re-patch anything that reverted before committing. |
| `task_rename_unit_test` | `unit_test` | mandatory `from_unit_test_id`, `to_unit_test_id` | Rename. |
| `task_delete_unit_test` | `unit_test` | mandatory `unit_test_id` | Delete. |
| `task_activate_unit_test` / `task_deactivate_unit_test` | `unit_test` | none | Toggle `active` — prefer deactivate over delete for temporarily-broken tests. |
| `task_import_unit_test_data_set` | `unit_test_data_set` | mandatory `from_unit_test_id` | Clone an existing data set (with its actual mock rows) onto this test — the only API path to populated mock data. |
| `task_from_unit_test_data_set_to_data_set_via_unit_test_modeler` | `unit_test_data_set` | none | Opens the Software Factory "Mock data" screen — the manual path to hand-author mock rows. |
| `task_show_functionality` | `unit_test` (via its control procedure) | none | Opens the Functionality screen for the control procedure under test. |

## What was NOT found

- **No queryable `test_case`/`test_step`/`test_scenario`/`smoke_test` API.** These exist only as
  members of the global `type_of_object` enum (`test_case=243, test_case_control_proc=272,
  test_case_run=539, test_step=256, test_step_check=260, test_step_result=543, test_step_run=544,
  test_suite=258, test_scenario=883, test_scenario_step=888, test_scenario_run=885,
  test_scenario_run_step=886, test_scenario_attachment=884, test_scenario_status=887, smoke_test=968,
  smoke_test_step=969`), not as entity sets reachable through `search_capabilities`/
  `get_entity_definition` in any of the 18 domains checked. This appears to be a recorded-GUI-test /
  smoke-test feature parallel to `unit_test`, but it is not exposed through this connector — don't
  build a skill workflow around it; confirm live on a different connector before assuming it's usable
  there.
- ~~No `data_set_tab`/`data_set_col`/`tab_data`/`tab_data_col` entity sets~~ — **retracted, see "Mock
  data" above.** These are real, writable entity sets; the earlier finding was a permissions artifact,
  not a genuinely missing feature.
