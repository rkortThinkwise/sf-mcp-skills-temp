# Drag-and-drop

## What it is

A Thinkwise drag-drop interaction is an ordinary table task, invoked by a drag gesture instead of a
button — there is no separate "drag-drop business logic" type. Three entities carry the whole
configuration, all confirmed live:

| Entity | Key | Role |
|---|---|---|
| `drag_drop` | `model_id, branch_id, drag_tab_id, drop_tab_id, drop_task_id` | The link definition itself: one row per (source table, target table, task) triple. |
| `drag_drop_parmtr` | `…, drag_tab_id, drop_tab_id, drop_task_id, drop_task_parmtr_id` | Column-to-parameter mappings for that triple — shared across every variant combination of it. |
| `drag_drop_matrix` | `…, drag_tab_id, drop_tab_id, drop_task_id, primary_key` | One row **per source/target variant combination** (including the plain table/table combo), independently enable-able. |

**This is a different mechanism from Scheduler activity dragging and map marker dragging** — see the
cross-references at the top of this skill. `drag_drop*` is specifically for dragging rows from one grid
(or tree/scheduler) onto another table/target task.

## Field reference — verified

`drag_drop`:

| Field | Type | Purpose |
|---|---|---|
| `drop_date_time_task_parmtr_id` | lookup → task parameter | Set only when the target is a Scheduler — the dropped time cell's date/time auto-populates this parameter (Platform 2026.2+; see `thinkwise_software_factory_scheduler_component`). |
| `drag_drop_effect` | enum: `copy_effect`=0, `link_effect`=1, `move_effect`=2 | Cursor/visual effect. **Verified still live and actively used** (not a legacy-only field) — real rows use both `copy_effect` (dragging a task onto a scheduler to *create* an activity) and `move_effect` (reordering rows in place). The task's actual behavior must still match whatever effect is shown — the effect is cosmetic, not enforcement. |

`drag_drop_parmtr`:

| Field | Type | Purpose |
|---|---|---|
| `drag_col_id` | lookup → `col` (source table) | Which column on the dragged row supplies the value. |
| `drop_task_parmtr_id` | lookup → task parameter | Which task parameter receives it. |
| `drop_behavior` | enum: `set_value`=0, `check_equal`=1 | `set_value` writes the dragged column's value into the parameter. `check_equal` instead validates that the parameter (typically already defaulted from the *target* row's context) equals this source column's value — a mismatch blocks the drop. This is exactly the mechanism behind "if a parameter is mapped from both source and target, the UI compares them and blocks on mismatch" — model a compatibility check (company, warehouse, parent ID) as a `check_equal` mapping. |

`drag_drop_matrix`:

| Field | Type | Purpose |
|---|---|---|
| `drag_tab_variant_id` / `drop_tab_variant_id` | lookup → `tab_variant` | The specific variant combination this row governs; both blank = the plain table-to-table combo. |
| `drag_drop_status` | enum: `drag_drop_disabled`=0, `drag_drop_enabled`=1, `drag_drop_other`=2 | Per-combination enablement. Bound tasks `task_enable_drag_drop`/`task_disable_drag_drop` flip this. |

`tab.drag_drop_default_enabled` (subject-level, not part of the `drag_drop*` family): whether the
source subject starts in drag mode by default in components that support it.

**Verified live examples**:

- `INSIGHTS`: `open_hours_per_customer` → `sales_invoice` via task `generate_sales_invoice_dnd`
  (`drag_drop_effect = move_effect`), mapping `customer_id` → `customer_id` with `drop_behavior =
  set_value`. The matrix shows this combination is **disabled** for the plain table/table pair and for
  most variant pairs, but explicitly **enabled** for `drop_tab_variant_id = concept_invoice` — a
  real-world case of "enable only the one variant pair where this interaction is actually valid," per
  the modeling guidance below.
- `PROJECT_MANAGER`: `project_task` → `project_planning_scheduler` via
  `project_planning_scheduler_add_activity` (`drag_drop_effect = copy_effect`) — this is the exact
  external-drag-onto-scheduler pattern documented in `thinkwise_software_factory_scheduler_component`,
  cross-confirmed from this side of the mechanism.
- `SQLSERVER_SF` (the Software Factory's own model): several same-table links such as `col` → `col`
  via `move_col_card_list_order_no` and `action_bar` → `action_bar` via a reorder task
  (`drag_drop_effect = move_effect`) — this is how the Software Factory itself implements "drag a row
  to reorder it," confirming that pattern for same-subject reordering generally.

## Step by step

1. Create or reuse a task that names the **business** operation (`assign_job_to_machine`,
   `add_stock_to_delivery`) — never a task named only `drag_drop`/`move`.
2. Create the `drag_drop` row: source tab, target tab, the task. Set `drag_drop_effect` to match what
   the task actually does. If the target is a Scheduler, set `drop_date_time_task_parmtr_id`.
3. Add `drag_drop_parmtr` rows: map the dragged row's identifying/required columns with
   `drop_behavior = set_value`; map any value that must match between source and target context with
   `drop_behavior = check_equal` (company, warehouse, parent ID, order).
4. In `drag_drop_matrix`, enable exactly the source/target variant combinations where the interaction
   is actually valid — everything starts `drag_drop_disabled`; don't blanket-enable every variant
   combination just because it's technically possible.
5. Decide `tab.drag_drop_default_enabled` for the source: on only when dragging is the screen's primary
   interaction (a dedicated planner/selection screen); off when users mostly select rows for other
   purposes. If it's not obvious which applies, ask the user rather than defaulting to off.
6. Confirm the Universal UI deployment setting `enableDragDrop` is `true` (an environment-level
   prerequisite, separate from all the model configuration above).
7. Choose the smallest refresh scope that stays correct (None/Row/Subject/Document) — for
   available/selected-list and planning-board patterns, Document is often the only fully safe choice
   but test whether Subject already suffices.

## Best practices and pitfalls

- **The task must revalidate everything itself** — source/target existence and state, authorization,
  tenant/company match, duplicate relationships, available quantity/capacity. The drag-drop link is a
  convenience layer, not a trust boundary; the same task can be invoked through Indicium, a process
  flow, or a plain button.
- **Multiple dragged rows fire the task once per row**, not batched — each can succeed or fail
  independently; make every operation idempotent and don't assume execution order.
- **`check_equal` is the model-level way to block incompatible drops** — prefer it over hoping users
  notice a mismatch after the fact.
- **Cursor effect must match real behavior** — a `move_effect` task that actually just creates a link
  and leaves the source untouched is misleading.
- **Always provide a non-drag alternative** (a button/task doing the same thing) — drag-and-drop has
  low discoverability and is difficult for keyboard/touch/motor-impaired users.
- **Don't enable every variant combination by default** — the matrix defaults every combination to
  disabled for a reason; enabling only the combinations that make business sense (as `INSIGHTS` does
  for `concept_invoice`) keeps the interaction matrix legible.
