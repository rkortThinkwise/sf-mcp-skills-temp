---
name: thinkwise-software-factory-unit-tests
description: Reference guide and workflow for suggesting and creating Thinkwise Software Factory unit tests (unit_test, data_set, and their child entities) via an MCP connector with Software Factory access. Use whenever a control procedure needs test coverage, or the user asks to suggest, review, or generate unit tests. Drives a propose → confirm → build process: analyze the target's logic, propose a scenario list grounded in its actual branches, get explicit sign-off, then build.
---

# Thinkwise Software Factory Unit Tests

A **unit test** (`unit_test`) proves one business rule with controlled input and an explicit expected
result — a default, layout, context, badge, task, process, subroutine, or insert/update/delete
handler/trigger, run against small mock data, checked against an expected output, message, or
assertion query. It is a Software Factory model object (Business Logic → Functionality → **Unit
tests** tab / Quality → Unit tests), version-controlled and deployed like any other control-procedure
artifact — not a hand-run SQL script and not a GUI recording.

**Don't confuse this with other Thinkwise testing mechanisms** — see "Unit test or another test?"
below. In particular: `sample_data_set` (`manage_datamodel` domain) is unrelated — domain-level example
values, not test mock data. A separate "recorded GUI test case" feature exists only as unexposed
metamodel concepts (`test_case`, `test_scenario`, `smoke_test`, …) in most connectors — confirm live
before assuming it's reachable; don't build around it if it isn't (see
`references/entity_reference.md`).

## The three-phase process

This skill is a workflow, not just an entity reference. Follow all three phases in order — never skip
straight to phase 3.

1. **Propose** — read the target's actual logic (generated SQL, branches, validations, messages) and
   turn it into a scenario list in plain language. Ground every scenario in something the code
   actually does; don't invent hypothetical edge cases the logic doesn't contain. See
   `references/test_design_guide.md` for the full decision framework (which test type, what to assert,
   how to size the mock data).
2. **Confirm** — present the scenario list to the user before writing anything. Each scenario states
   the condition and the expected result in natural language, no SQL. Wait for explicit approval; treat
   feedback as a revision loop, not a one-shot. Never generate model objects before this sign-off —
   the `thinkwise_software_factory_mcp_base` "Confirm-before-mutate" convention applied at this skill's
   own grain.
3. **Build** — once approved, create the `unit_test` row(s) and their input/output/filter/message
   children via the connector's staging flow (`stage_resource` → `patch_resource` → `commit_resource`,
   or `stage_task` for lifecycle tasks). See "Building a unit test" below for entity order and the
   mock-data gap.

## Domain and entity discovery

Unit-test authoring lives in a domain confirmed live as **`manage_unit_tests`** — try this directly
first via `get_entity_definition`/`search_domain_capabilities`; only fall back to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection. This skill is connector-agnostic: it names entities, fields, and enums, not any one
connector's literal tool names. Confirmed live: `manage_control_procedures`, `manage_datamodel`, and
`manage_cubes` do **not** additionally expose unit-test authoring entities — `manage_unit_tests` is
where all of it lives, alongside a `manage_datamodel`-level `tab`/`task`/`subroutine`/`process_flow`
lookup for finding the object under test in the first place.

**Core entities** (full field tables, enums, and tasks in `references/entity_reference.md`):

| Entity | Role |
|---|---|
| `unit_test` | The test definition — one row per test. Type, target object, flags, prep/assertion query, status. |
| `unit_test_col_input` / `_col_output` | Default **and** insert/update/delete handler test: input column values / expected output values+condition — `HANDLE_INSERT`/`UPDATE`/`DELETE` write through this same entity, confirmed live, not a separate `unit_test_col_handler`. |
| `unit_test_col_type` | Layout/context test: expected field type or mandatory state. |
| `unit_test_col_filter` | Row-filter test (update/delete statement): which rows the filter should match. Also how a `TRIGGER_INS`/`UPD`/`DEL` test targets one existing row, since a primary-key column is rejected as an ordinary input on that type. |
| `unit_test_task_parmtr_input` / `_output` | Task test: input parameter values / expected output+condition. |
| `unit_test_subroutine_parmtr_input` / `_output` | Subroutine/function test: same shape, keyed by subroutine parameter. |
| `unit_test_process_variable_input` / `_output`, `_process_step_input` / `_output` | Process test: process variable and step-level checks. |
| `unit_test_ref_output` | Context test: expected reference/detail-tab availability (output-only). |
| `unit_test_tab_report_output` / `_tab_task_output` | Context test: expected report/task availability (output-only). |
| `unit_test_fixed_parmtr_input` / `_output` | Fixed parameters shared across types — chiefly `cursor_from_col_id` for defaults. |
| `unit_test_msg` | Expected message(s) for the sad-flow / validation path. |
| `unit_test_query` | Per-`rdbms_type` override of `preparation_query`/`assertion_query` on a multi-dialect model. |
| `data_set` / `unit_test_data_set` | Named mock-data bundle and its link to one or more unit tests. |

**No `$expand` chain from `unit_test`.** All the `unit_test_col_*`/`*_parmtr_*`/`*_process_*` child
entities relate to `unit_test` only by shared composite key (`model_id, branch_id, unit_test_id, ...`)
— query them directly with `execute_odata_query` filtered on `unit_test_id`, don't expect a navigation
property to pull them in.

## Building a unit test

1. **Identify the target precisely** — the exact `control_proc_id` (and `tab_id`/`task_id`/
   `report_id`/`process_flow_id`/`process_action_id`/`subroutine_id` depending on type) driving the
   logic under test. Read the generated code (see `thinkwise_software_factory_create_control_procedures`
   for how to find the real control procedure behind an object) so the scenario list in phase 1 is
   grounded in what the code actually branches on, not a guess. **Before scoping candidates by scanning
   generated program objects for a code type (e.g. every table with a Default enabled), filter
   `control_proc` by `control_proc_type` instead** (see
   `thinkwise_software_factory_create_control_procedures`): `program_object_item` (`1`) is hand-written
   logic for one specific object and is what's actually worth testing; `program_object` (`0`) and
   `meta_definition` (`2`) are shared generators/framework infrastructure that many generated objects
   point at without containing any object-specific logic of their own. Scanning generated objects
   first and treating every one as a distinct test target overcounts wildly — a project can show
   dozens of generated Default program objects while only a couple actually contain custom logic.
   **If more than one `control_proc_id` could plausibly be the target** — e.g. several Default
   `program_object_item` rows exist on the same table, or the user's description doesn't pin down which
   handler/trigger they mean — stop before reading any of their code or drafting scenarios. List the
   candidates with enough context to tell them apart (table, code type, trigger event — e.g. "HANDLE_UPDATE
   on `opportunity`, fires on any column change" vs. "TRIGGER_UPD on `opportunity`, cascades to
   `activity`") and ask the user to confirm which one is the actual target before proceeding.
2. **Create the `unit_test` row.** Key: `unit_test_id` (purpose-driven, snake_case — see naming below).
   **`unit_test.model_id`/`branch_id` are hidden and reject direct writes on a fresh top-level insert,
   and a dependent add under `control_proc` (which does resolve that context) has still been found to
   reject `type_of_object` as an unknown property despite showing it as editable.** The reliable path,
   confirmed across every type tried — **copy any existing `unit_test` row via `task_copy_unit_test`
   and override its target fields in the same call** (`to_unit_test_id`, `unit_test_type_id`,
   `type_of_object`, the target field, `control_proc_id`) — see `references/entity_reference.md` for the
   full mechanics and why the dependent-add path falls short. After the copy, check for stray copied
   input/output rows before adding new ones (a column name shared between the source and target table
   for unrelated reasons can carry a row across even when the object types differ).
   Set `unit_test_type_id` (`DEFAULT`, `LAYOUT`, `CONTEXT`, `BADGE`, `TASK`, `PROCESS`, `SUBROUTINE`,
   `HANDLE_INSERT`/`HANDLE_UPDATE`/`HANDLE_DELETE`, `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL` — confirm
   the full list live via `unit_test_type`, don't assume this is exhaustive) and `type_of_object` (the
   *target kind* — Table, Task, Process flow, Subroutine, … — a large global enum; confirm the right
   value via `unit_test_type_object` rather than guessing, though `0`=Table, `11`=Task, `30`=Process
   flow, and `231`=Subroutine have been observed repeatedly across real models). Fill the one scalar
   key matching that kind (`tab_id` for table-scoped types, `task_id` for `TASK`, `subroutine_id` for
   `SUBROUTINE`, `process_flow_id`/`process_action_id` for `PROCESS`, `report_id` for a report-context
   test) plus `control_proc_id`, a full-sentence `unit_test_description` (never leave this vague — see
   naming below), and `active = true`.
3. **Add input rows** for every column/parameter the scenario controls and **output rows** for every
   value being asserted. The entities in the table above (`unit_test_col_input`/`_col_output`,
   `unit_test_task_parmtr_input`/`_output`, …) describe the underlying storage and are correct for
   *reading* an existing test back, but a direct add on them is rejected outright — **write through
   `unit_test_input_parmtr_overview`/`unit_test_output_parmtr_overview` instead**, added as a dependent
   record under `unit_test`. Set `parmtr_id` to the column/task-parameter/subroutine-parameter/
   fixed-parameter id (e.g. `cursor_from_col_id`) and the row resolves which underlying child entity it
   really is. See `references/entity_reference.md` for the full mechanics, including that leaving a
   column at its natural blank/null state means *not creating a row for it at all* (input rows are
   mandatory-valued, not nullable placeholders). Each output row gets a `unit_test_condition`
   (16-value enum — `equal_to`, `smaller_than`,
   `greater_than_or_equal_to`, `between`, `contains`, `is_empty`, … — full list in the reference file;
   note it's **not** the 8-value set a naive guess produces, e.g. `smaller_than`/`is_empty`, not
   `less_than`/`is_null`). For a default that only reacts when a specific column changed, add a
   `unit_test_fixed_parmtr_input` row with `unit_test_parmtr_id = 'cursor_from_col_id'`.
   **Before typing a placeholder/sentinel value for an id-shaped column or parameter** (one ending
   `_id`, e.g. an owner/contact/foreign-key reference), confirm the backing domain's actual data type
   first — many id domains are numeric (`int`), not string, despite the naming convention; a string
   sentinel like `'test_emp_sentinel'` is rejected outright where the real domain is numeric.
   **For a `TRIGGER_INS`/`TRIGGER_UPD`/`TRIGGER_DEL` test, identify the row via `unit_test_col_filter`
   instead of an ordinary input row** — a primary-key column is rejected as a regular input on this
   type. `unit_test_col_filter` is directly addable at the top level (unlike the overview entities
   above); see `references/entity_reference.md`.
4. **Add `unit_test_msg` rows** for any expected validation/warning message on the sad-flow path.
   **Confirm the referenced `msg_id` actually exists first** — control-procedure code can call a
   message id that was never created (a real bug to flag, not something to build around silently); see
   `references/test_design_guide.md`'s "Expected messages" section.
5. **Link mock data — only if the logic actually needs it.** A test whose control procedure only
   reads/writes the cursor row's own columns (the common case for a simple default expression) needs no
   `data_set` at all — the input/output rows from step 3 are sufficient; skip straight to step 6. Only
   when the scenario depends on other rows/tables existing: create/reuse a `data_set` row, link it via
   `unit_test_data_set` — **if linking a data set to a test that has no existing link fails outright
   (not a permissions issue), fall back to `preparation_query`/`assertion_query` with plain SQL instead
   of this pipeline for that test**, remembering to wrap any explicit identity-column values in
   `SET IDENTITY_INSERT <table> ON`/`OFF` — then follow **`references/mock_data_guide.md`** for the full, confirmed-working
   step-by-step recipe (create the data set → `task_create_data_set_tab` per table, in FK dependency
   order → select columns to mock via `task_data_set_col_selection_select` → `task_create_tab_data` per
   row → set values via `tab_data_col_overview`). That guide also covers: `task_import_unit_test_data_set`
   for cloning an existing compatible data set instead of building from scratch; always setting
   identity/PK columns to an explicit constant value rather than leaving them auto-generated, so tests
   reference known, stable ids; mocking the full FK chain bottom-up when a table has mandatory foreign
   keys; and a troubleshooting table for the connector's rough edges (a raw insert on `data_set_tab`
   reliably 500s with a spurious duplicate-key error — always use the bound task instead; the enum field
   `type_of_data_value` needs its numeric code via `value_kind:"data"`, not the display label). If every
   permission needed turns out to be missing and there's truly no existing data set to clone from, the
   Software Factory UI's **Mock data** screen (reachable from `unit_test_data_set` via the bound task
   `task_from_unit_test_data_set_to_data_set_via_unit_test_modeler`) remains the manual fallback — but
   confirm permissions first rather than assuming this is required.
   Design the mock dataset itself per `references/test_design_guide.md`'s "Designing mock datasets"
   section — smallest set that satisfies the condition, named subject/supporting/contrasting/sentinel
   rows rather than arbitrary numeric ids, no dependence on whatever happens to already be in a dev
   database. Favor a small scenario-specific dataset over growing a large shared one — a dataset with a
   generic name (`general`, `test_data`) reused across many unrelated tests is a maintenance risk, not a
   shortcut. **Whenever this step invents concrete values — boundary numbers, sentinel row names, or the
   actual data behind a named subject/supporting/contrasting/sentinel row — list those concrete values
   alongside the scenario description at phase 2 and get them approved too.** Approving a scenario worded
   as "quantity exceeds max" is not the same as approving that `max` was set to `100` — surface the
   specific numbers/names chosen so the user can catch a wrong boundary or a misleading sentinel value
   before the test is built, not after it fails.
6. **Multi-dialect models**: if `branch_rdbms_type` (see
   `thinkwise_software_factory_create_control_procedures`) returns more than one row, the generic
   `unit_test.preparation_query`/`assertion_query` may not be portable — add a `unit_test_query` row
   per `rdbms_type` needing different SQL, same pattern as `control_proc_template` per dialect.
7. **Run it.** `task_add_job_to_test_unit_test_single` (bound to `unit_test` or
   `test_unit_test_overview`) executes one test; `task_add_job_to_test_unit_test` runs the whole active
   suite. Poll `test_unit_test`/`test_unit_test_overview` (`job_id`, status enum
   `scheduled=0/executing=1/wait_for_user=2/successful=3/failed=4/cancelled=5/aborted=6/warning=7/info=8`)
   or re-read `unit_test.result_status` (`successful=0/model_mismatch=1/failed=2/not_executed=4/
   executing=5/scheduled=6` — note this is a **different** enum, don't conflate the two) for the
   outcome. **A `failed` (not `model_mismatch`) result on an `equal_to` datetime *or* money/decimal
   output that looks logically correct is usually a string-format mismatch, not a logic bug** — see the
   datetime/decimal note in `references/entity_reference.md`; check `unit_test_result_parmtr.output_value`
   for the actual raw string the run produced. **A `failed` result despite the message/assertion check
   itself reporting success can also mean a `should_rollback` mismatch** — its outcome tracking is only
   confirmed reliable for `HANDLE_UPDATE`; default it to `false` for `TRIGGER_*`/`TASK`/`HANDLE_INSERT`/
   `HANDLE_DELETE` tests (see `references/entity_reference.md`). `task_show_unit_test_code` (param `rdbms_type`) previews the generated test SQL without
   running it — useful to sanity-check a test before the first run. `task_cancel_job` stops a running
   batch.
8. **Lifecycle housekeeping**: `task_copy_unit_test` (clone the definition, not the mock data link, to a
   new id — also the primary creation mechanism per step 2 above; re-check copied input/output rows
   after a cross-table copy), `task_rename_unit_test`, `task_delete_unit_test`, `task_activate_unit_test`/
   `task_deactivate_unit_test`. Prefer deactivating a temporarily-broken test over deleting it — see
   "Recommended team standard" in the design guide.

**AI-assisted first draft**: `task_enrichment_generate_unit_test` (bound to `unit_test`; mandatory
`description`, `unit_test_type_id`, `type_of_object`; optional `tab_id`/`task_id`/`report_id`/
`process_flow_id`/`process_action_id`/`subroutine_id`/`control_proc_id`) asks the Software Factory's own
AI to draft a test from a natural-language description. Useful as a starting point for phase 3, but
still review the generated scenario against the actual code before treating it as covering what phase 1
proposed — it's a drafting aid, not a substitute for the propose/confirm steps.

## Naming and description conventions

- **`unit_test_id`**: `{object_id}_{condition_and_expected_result_in_snake_case}` — describes the
  scenario, not just which object is under test (the object is already in `control_proc_id`/`tab_id`/
  `task_id`). Avoid restating "test"/"unit test" in the id.
- **`unit_test_description`**: a full sentence stating the condition and the expected result — e.g.
  *"Exceeding the maximum number of packaging layers produces the max-layers warning."* **Never** leave
  this empty or generic (`"test voor demo"`, `"test"`) — this is the single most common quality gap
  found across real models (see `references/test_design_guide.md`) and it's what makes a failing test
  meaningful to whoever reads the failure six months later.
- One test = one recognizable business rule. If a scenario needs "and also check this unrelated thing,"
  split it into two tests.

## Pre-flight checklist

- **Never build before the user has approved the scenario list.** Phase 1 output is plain language, no
  SQL, no model writes.
- **After building a test, actually run it (`task_add_job_to_test_unit_test_single`/
  `task_add_job_to_test_unit_test`) and confirm the result before considering the work done.** A
  Software Factory validation flags any unit test that's been modeled but never executed — "built" and
  "passing" are not the same claim, and a run blocked by an outage/stale deploy is still an open item
  to flag, not a finished task.
- **After building or refactoring tests, check whether a `data_set` ended up with no `unit_test_data_set`
  link at all.** An orphaned data set is flagged by a Software Factory validation — delete it or link
  it to a test rather than leaving it behind.
- **Identify the exact `control_proc_id`/target object before writing any test** — read the generated
  code, don't guess which branch a vague description refers to.
- **`unit_test_condition` has 16 values, not 8** — `smaller_than`/`smaller_than_or_equal_to`/
  `is_empty`/`is_not_empty`, not `less_than`/`less_than_or_equal_to`/`is_null`/`is_not_null`. Confirm
  against `references/entity_reference.md` rather than guessing from habit.
- **`type_of_object` is a large global enum, not the same thing as `unit_test_type_id`.** The latter
  says *what kind of test* (`DEFAULT`, `TASK`, …); the former says *what kind of object* is under test
  (Table, Task, Process flow, Subroutine, …) and picks which scalar key field on `unit_test` is
  populated. Confirm the value live via `unit_test_type_object` for anything beyond the four commonly
  observed (`0`=Table, `11`=Task, `30`=Process flow, `231`=Subroutine).
- **No child entity is reachable from `unit_test` via `$expand`.** Query every `unit_test_col_*`/
  `*_parmtr_*`/`unit_test_msg` row directly, filtered by `unit_test_id`.
- **Mock data rows ARE writable through this API**, given the right table-level permissions on
  `data_set`/`data_set_tab`/`tab_data` — see `references/mock_data_guide.md` for the full recipe. Don't
  default to "manual Studio step required" without first checking `get_entity_definition` on `tab_data`
  for `allow_add`/`bound_tasks` (a missing `task_create_tab_data` there is the actual signal a
  permission grant is needed) — and don't claim a test is fully built if its data set has no rows and
  no path was taken to populate it.
- **Don't confuse `unit_test.result_status` (per-test last-run outcome) with the `test_unit_test`
  job-status enum (per-batch execution progress)** — different enums, different meanings, checked at
  different points in the run flow.
- **Don't build against `sample_data_set`** — it's an unrelated `manage_datamodel` feature (example
  values for a domain), not unit-test mock data, despite the similar name.
- **A "recorded GUI test case" concept (`test_case`/`test_scenario`/`smoke_test`) may exist only as
  metamodel enum members, not a queryable API** — confirm live before promising it; if it isn't exposed,
  say so rather than building toward it. See `references/entity_reference.md`.
- **Multi-dialect model?** Check `branch_rdbms_type` before assuming one `preparation_query`/
  `assertion_query` covers every platform — add `unit_test_query` overrides per `rdbms_type` as needed.
- **Every test needs a real description and a real name** — the single most common defect in existing
  models is a vague or empty `unit_test_description`.
- **Cover both the happy and sad path** for anything that validates or branches — a test proving "no
  database error occurred" without checking what changed (or didn't) is not sufficient.
- **Don't default to one large shared `data_set`.** A dataset spanning dozens of tables/hundreds of
  columns (real examples exist) makes every attached test fragile to unrelated edits — see "Designing
  mock datasets" in `references/test_design_guide.md` for sizing and reuse guidance.
- **Create new tests by copying an existing one via `task_copy_unit_test`, not by adding `unit_test`
  directly or as a dependent record under `control_proc`** — the direct paths have been found to reject
  `type_of_object`. See step 2 above and `references/entity_reference.md`.
- **Default `should_rollback` to `false` outside `HANDLE_UPDATE`.** Its outcome tracking isn't reliable
  for `TRIGGER_*`/`TASK`/`HANDLE_INSERT`/`HANDLE_DELETE` — rely on `should_abort` plus message/assertion
  checks for those types instead.
- **An explicit value in `preparation_query` for an identity-typed primary key needs
  `SET IDENTITY_INSERT <table> ON`/`OFF`** around that table's insert, or it's rejected outright.
- **Re-read `preparation_query`/`assertion_query` after patching `use_preparation_query`/
  `use_assertion_query`** — toggling either flag can reset the other query-text field as a side effect.
- **A primary-key column is rejected as an ordinary input on `TRIGGER_INS`/`UPD`/`DEL` tests** — target
  the row via `unit_test_col_filter` instead (directly addable, unlike the input/output overview
  entities).
