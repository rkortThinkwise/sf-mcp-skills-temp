# Mock data step-by-step guide (`data_set` → `tab_data`)

Confirmed live end-to-end against `sf_mcp`, domain `manage_unit_tests` (2026-08-06, model
`RK_SCHEDULER_TEST`), after the connecting user granted table-level permissions on `data_set`,
`data_set_tab`, and `tab_data`. **This supersedes any earlier claim in this skill family that mock rows
are "not writable"/"API-blocked"/"manual Studio step only" — that conclusion was a permissions gap on
one connector/user, not a platform limitation.** Every entity below has a working API path once granted.
If you hit a 403 or a field stuck in `hidden`/`readonly` state that this guide says should be editable,
suspect a missing grant first (see "Permissions" at the end) before falling back to a manual step.

**Before starting this recipe on a brand-new test with no existing `data_set` link**: linking a data set
to a unit test that doesn't have one yet is permission-gated, same as every other write in this guide —
it is **not** a separate structural dead end. Stage an empty add on `unit_test_data_set` first and check
whether `model_id`/`branch_id` come back `hidden` (missing grant, see "Permissions" below) or `editable`.
If editable, patch all four key fields (`model_id`, `branch_id`, `unit_test_id`, `data_set_id`) together
in one call — confirmed live to commit cleanly once the grant is in place. Patching only
`unit_test_id`/`data_set_id` while `model_id`/`branch_id` are still unset 403s regardless of which
property is patched first, which looks like a deeper bug but is just the same missing-context symptom.
**Adding it as a dependent record under either `unit_test` or `data_set` is a separate, still-broken
path** (a detail-navigation-not-found error) unaffected by permissions — the relationship is exposed
under a qualified/compound target name the connector's own parent-detection can't resolve; don't retry
that route. If the grant genuinely can't be obtained, fall back to `preparation_query`/`assertion_query`
with plain SQL instead (see `entity_reference.md`'s `unit_test` field table), remembering to wrap any
identity-column inserts in `SET IDENTITY_INSERT <table> ON`/`OFF`. This pipeline remains the right choice
once a data set is successfully linked, or when cloning via `task_import_unit_test_data_set` onto a test
that already has the link.

## Entity map

| Entity | Role | Key |
|---|---|---|
| `data_set` | The named mock-data bundle itself. | `model_id, branch_id, data_set_id` |
| `data_set_tab` | Which table is mocked within a data set. | `+ tab_id` |
| `data_set_col_selection` | Per-column on/off switch for mocking, auto-created (one row per real column) the moment `data_set_tab` is created. | `+ tab_id, col_id` |
| `data_set_col` | Selected-column bookkeeping entity — exists but you don't normally touch it directly; `data_set_col_selection` is the one with the select/deselect tasks. | `+ tab_id, col_id` |
| `tab_data` | One mock row. | `+ tab_id, row_id` (`row_id` is server-assigned on create) |
| `tab_data_col_overview` | The actual value of one column on one mock row. Auto-created — one row per *selected* column — the moment `tab_data` gets created. | `+ tab_id, row_id, col_id, rdbms_type` |
| `unit_test_data_set` | Junction linking a data set to the unit test(s) that use it (nav'd from `unit_test` as `.mock_data`). | `unit_test_id, data_set_id` |

None of `data_set_tab`/`data_set_col_selection`/`tab_data`/`tab_data_col_overview` expose a `$expand`
nav chain down from `data_set` — query each directly, filtered on `data_set_id` (+ `tab_id`/`row_id` as
you go deeper).

## Step 0 — resolve context

Pin `model_id`/`branch_id` per `thinkwise_software_factory_mcp_base`. Get `rdbms_type` from
`branch_rdbms_type` (usually a single row; `0`=sqlserver, `1`=iseries, `3`=oracle, `4`=postgresql) — you
need this value for every `tab_data_col_overview` key.

## Step 1 — create the `data_set` header (if it doesn't exist yet)

Try a direct add first:

```
stage_resource(action=add, entity_set=data_set, properties=[{data_set_id: "..."}])
```

- If `model_id`/`branch_id` come back `editable` in the staged fields, just patch them (`model_id` then
  `branch_id` — patch separately, not always safely combined) and commit. This is a normal top-level
  insert once permissions allow it.
- If they come back `hidden` and mandatory, the add will 400/422 (`gui_cannotsavemandatory: Model`) no
  matter what you try — that's the field-permission gap, not a missing feature. In that case, use
  `task_copy_data_set` (bound to any *existing* `data_set` row; params `from_data_set_id`/
  `to_data_set_id` — `from_data_set_id` comes back readonly and auto-fills) to clone a donor into a new
  id. This needs at least one existing `data_set` row in the model to clone from; if there are zero and
  permissions aren't fixable, that first row is a genuine manual-Studio-step gap.

## Step 2 — add each table you need to mock

```
stage_task(task_name="task_create_data_set_tab", entity_set="data_set_tab",
           parent_entity_set="data_set", parent_key={model_id, branch_id, data_set_id},
           properties=[{tab_id: "..."}])
→ commit_resource
```

**Always use this bound task, never a raw `stage_resource(action=add)` on `data_set_tab`.** A raw add
reproducibly fails with `mssql_error_duplicate_key` regardless of `data_set_id`/`tab_id` — even against
an empty table with zero existing rows anywhere in the model. This looks like a bug specific to the
raw-insert code path; the task avoids it entirely and always works.

If the table has mandatory foreign keys to other tables (e.g. `opportunity.account_id` → `account`,
`account.account_owner_employee_id` → `employee`), **add those referenced tables too, and do this whole
recipe for them first, bottom-up in FK dependency order** — see "Referential integrity" below. Repeat
this step once per table in the chain.

Creating a `data_set_tab` row auto-generates one `data_set_col_selection` row per real column on that
table, with the primary key column(s) pre-selected (`selected=true`) and everything else off.

## Step 3 — select which columns to mock

Query current state first:

```
execute_odata_query: /data_set_col_selection?$filter=model_id eq '...' and branch_id eq '...'
  and data_set_id eq '...' and tab_id eq '...'&$select=col_id,selected,mand
```

Check `mand` (mandatory) per column from `col` in `manage_datamodel` if not already visible. **Every
mandatory column must be selected**, or the row you create in Step 4 has no value to give it and the
mock insert will be incomplete/wrong. Optional columns only need selecting if the scenario actually
reads them.

For each column to enable:

```
stage_task(task_name="task_data_set_col_selection_select", entity_set="data_set_col_selection",
           key={model_id, branch_id, data_set_id, tab_id, col_id})
→ commit_resource
```

No parameters — the task just flips the switch on the bound record. `task_data_set_col_selection_deselect`
is the inverse, for removing a column later.

## Step 4 — create the row(s)

```
stage_task(task_name="task_create_tab_data", entity_set="tab_data",
           parent_entity_set="data_set_tab",
           parent_key={model_id, branch_id, data_set_id, tab_id})
→ commit_resource
```

No parameters, returns nothing useful directly — query `tab_data` back afterward, filtered on
`data_set_id`/`tab_id`, to get the server-assigned `row_id`(s). Repeat once per row you need.

**This bound task only appears in `get_entity_definition`'s `bound_tasks` list for `tab_data` once
permissions are granted** — before that, `tab_data` shows `allow_add:false` and an empty `bound_tasks`
array, and any raw `stage_resource(action=add)` attempt 403s outright. If `task_create_tab_data` isn't
showing up, that's the signal to go ask for the permission grant, not to look for a different task name.

Creating a `tab_data` row auto-generates one `tab_data_col_overview` row per **currently-selected**
column (from Step 3), each initialized to `type_of_data_value = fallback_value` (uses the column's
model-level default, or null if none) with no value set.

## Step 5 — set the actual column values

For every column you want a real value for, on every row:

```
stage_resource(action=edit, entity_set="tab_data_col_overview",
  key={model_id, branch_id, data_set_id, tab_id, row_id, col_id, rdbms_type},
  properties=[
    {property: "type_of_data_value", value: "0", value_kind: "data"},
    {property: "value", value: "<literal>"}
  ])
→ commit_resource
```

Both properties can be set together safely in one `stage_resource`/`patch_resource` call — verify via
the returned `fields` echo, but the general "last-property-drop" quirk documented elsewhere in this
connector family did not reproduce here.

Critical details:

- **`type_of_data_value` must be patched with `value_kind: "data"` and the raw numeric string**
  (`"0"` = constant_value, `"1"` = expression, `"2"` = null_value, `"3"` = fallback_value). Passing the
  enum's display label (`"constant_value"`) directly 400s with `invalid_input: "The value for Type of
  value is invalid"`.
- `value` is always a plain string, regardless of the column's real data type. A date column accepted
  plain ISO format (`"2026-09-15"`) with no special SQL formatting needed — unlike the `assertion_query`
  datetime-equality quirk documented elsewhere in this skill, this is a normal UI-facing field, not a raw
  SQL literal.
- **Enum-domain columns** (e.g. a `tinyint` status/stage column backed by a Thinkwise domain with named
  elements) need the underlying **`db_value`** from `elemnt` (`manage_datamodel` domain, filtered by
  `dom_id`), not the `elemnt_id` label. E.g. for a `dom_id='opportunity_stage'` domain with elements
  `prospecting`/`qualification`/`proposal`/…, the actual `value` to write is `elemnt.db_value`
  (`"0"`/`"1"`/`"2"`/…), found via:
  ```
  /elemnt?$filter=model_id eq '...' and branch_id eq '...' and dom_id eq '...'
    &$select=elemnt_id,db_value&$orderby=order_no
  ```
- **Colour/hex-looking domains backed by a physical `int` type** (check the domain's data type in
  `manage_datamodel`'s `dom` entity before assuming a colour-sounding column takes hex-string input)
  need the plain **decimal integer** equivalent of the colour, not a `"#RRGGBB"` string — a hex-string
  value gets emitted unquoted into the generated insert SQL as an invalid token, which fails as a
  parse-time syntax error rather than a clean validation rejection. Convert the hex value to decimal
  first (e.g. `0x0000FF` → `255`).
- Bit/boolean columns: pass `"1"`/`"0"` as the value string.
- **Identity/PK columns default to `fallback_value` (auto-generated) and don't strictly need a value**
  — but see "Always mock identity values explicitly" below for why you almost always want to override
  them anyway. Setting an explicit `constant_value` on an identity column works with no special
  "identity insert" handling — same `stage_resource(edit)` pattern as any other column.

## Always mock identity/PK values explicitly

**Standing rule, not just a one-off preference**: set every identity/PK column to an explicit
`constant_value`, never leave it at the default `fallback_value`/auto-generated. Reason: a unit test's
assertions (row-filter conditions, expected-output values, FK references from other mocked rows) need to
name a *known, stable* id — if the real DB auto-assigns it, you can't predict what it will be, and the
test becomes non-deterministic or requires an extra round-trip just to discover the assigned id before
writing the rest of the test. Pick small, memorable, scenario-specific ids (not whatever the identity
sequence happens to be at) and reuse the *same* explicit ids consistently across every table in the FK
chain (see next section) — that's what makes the whole mock dataset legible at a glance.

## Referential integrity — mock the FK chain, not just the target table

If the table you're mocking has a mandatory FK to another table, **that other table needs a mocked row
too, with a matching explicit id** — otherwise a unit test that actually *executes* against this data
will hit a real FK constraint violation at run time (the row-creation/value-setting steps above will
still succeed with no error; the failure only shows up when the test runs).

Recipe:
1. Find every mandatory FK column on the table you care about (`col` in `manage_datamodel`, filtered by
   `tab_id`, `$select=col_id,mand`, cross-referenced against which columns are references — a `dom_id`
   of `id` combined with a `col_id` ending `_id` that isn't the table's own PK is a strong signal, but
   confirm the actual FK target via the table's `ref`/`ref_col` metadata if unsure).
2. Recurse: does the FK target table itself have mandatory FKs? Keep walking until you reach tables with
   no mandatory FK dependencies (leaf tables).
3. Mock bottom-up: leaf tables first, each with explicit PK values, then the next table up referencing
   those exact values, and so on until you reach the table you originally wanted.

Worked example from this session — mocking 3 `opportunity` rows required the full chain
`department` (1 row, PK=1) → `employee` (1 row, PK=1, `department_id`=1) → `account` (3 rows, PK=1/2/3,
`account_owner_employee_id`=1 for all) → `opportunity` (3 rows, PK=1/2/3, `account_id`=1/2/3,
`opportunity_owner_employee_id`=1). Every id was explicit and reused consistently, per the identity-value
rule above.

## Maintenance tasks

| Task | Bound to | Purpose |
|---|---|---|
| `task_delete_tab_data` | `tab_data` | Delete one mock row. |
| `task_copy_tab_data_values` | `tab_data` | Clone a row's values ("Copy row") — useful for near-duplicate rows, remember to then override the identity/PK column on the copy. |
| `task_data_set_col_selection_deselect` | `data_set_col_selection` | Turn off mocking for a column (its `tab_data_col_overview` rows presumably revert to fallback — re-verify if you rely on this). |
| `task_delete_data_set_tab` | `data_set_tab` | Remove a whole table from the data set (cascades to its rows/column selections — verify before relying on this for cleanup). |
| `task_copy_data_set` | `data_set` | Clone an entire data set (including tables/columns/rows) to a new id. |
| `task_rename_data_set` | `data_set` | Rename. |
| `task_delete_data_set` | `data_set` | Delete the whole data set. |
| `task_import_unit_test_data_set` | `unit_test_data_set` | Clone a *different* unit test's data set onto this one — still the right move for reusing an existing well-shaped dataset rather than rebuilding from scratch. |

## Permissions

Table-level permissions on `data_set`, `data_set_tab`, and `tab_data` are **independent grants** — a
403 or a hidden/readonly field on one does not imply the same on the others; check/ask about each
separately if something in this recipe isn't behaving as documented. Signals that a grant is missing
(vs. a genuine step you did wrong):

- `data_set` direct add: `model_id`/`branch_id` show as `state: "hidden"` in the staged `fields` response
  instead of `"editable"`.
- `tab_data`: `get_entity_definition` shows `allow_add: false` and `task_create_tab_data` is absent from
  its `bound_tasks` list.
- Any raw `stage_resource`/`stage_task` call 403s with `staging_request_failed`.

Once granted, re-run `get_entity_definition` on the affected entity to confirm the field states/task
list actually changed before retrying the write — the tool result reflects live permission state, not a
cached snapshot.

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `gui_cannotsavemandatory: Model` on `data_set` add | `model_id` hidden — permissions | Grant permission, or `task_copy_data_set` from a donor |
| `mssql_error_duplicate_key` on `data_set_tab` add, with zero existing rows | Bug in the raw-insert path | Use `task_create_data_set_tab` instead of raw add |
| `403 staging_request_failed` on `tab_data` add | Missing permission on `tab_data` | Grant permission; then use `task_create_tab_data` |
| `invalid_input: The value for Type of value is invalid` on `tab_data_col_overview` | Passed the enum label instead of the numeric code | Use `value_kind: "data"` with `"0"`/`"1"`/`"2"`/`"3"` |
| Test fails at run time with an FK violation, even though mock rows were created with no API error | FK target table wasn't also mocked | Walk the FK chain bottom-up, mock every referenced table with matching explicit ids |
