# Grid

## What it is, and when to use it

The Grid shows many rows in aligned, comparable columns — the default choice for cross-row comparison,
sorting/filtering/grouping, multi-row selection, and bulk actions. In a detail tab it limits itself to
the parent's related rows. Prefer **Form** for one rich record, **Card List** for scan-and-select on
small screens, **Treeview** for hierarchy/grouping navigation.

## Field reference — verified

`tab` / `tab_variant_overview` (subject-level behaviour):

| Field | Type | Purpose |
|---|---|---|
| `allow_edit_grid` | flag | Cell editing permitted at all. |
| `grid_default_editable` | flag | Auto-edit — Grid opens ready for cell edits (vs. deliberate edit-mode). |
| `allow_add_grid` | flag | Add-in-Grid via a pinned top row. |
| `allow_mass_update` / `allow_import` / `allow_export` | flags | Bulk operations. |
| `grid_row_height` | int (px) | Row height — raise for images/multiline content. |
| `auto_resize_grid_col` | enum: `no`=0, `headers_and_data`=1, `data_only`=2 | Auto column width strategy — set `no` on any table/variant with 20+ visible columns (documented performance guidance). |
| `no_of_fields_locked` | int | How many leading grid-ordered columns stay pinned while scrolling horizontally. |
| `grp_box_visibility` | enum: `never`=0, `when_grouped`=1, `always`=2 | Whether the row-grouping drop box shows. |
| `grp_grid_default_expanded` / `grp_grid_default_expanded_level` | flag / byte | Default state of runtime row grouping. |
| `enable_multirow_select` | flag | Multi-row selection (only one Grid can have it active at a time; on mobile it needs long-press). |
| `drag_drop_default_enabled` | flag | Whether this subject starts in drag mode by default (see § 5). |

`col` / `tab_variant_grid(_overview)` (per-column):

| Field | Type | Purpose |
|---|---|---|
| `grid_order_no` | int | Column order — may differ deliberately from Form order. |
| `grid_type_of_col` | enum: `editable`=0, `read_only`=1, `hidden`=3 | Capped by the underlying column definition. |
| `grid_col_width` | int (px) | Explicit width override. |
| `grid_field_in_next_grp` / `grid_next_grp_label` | flag / string | **Header group** — a visual column-heading group (e.g. "Planned"/"Actual"), distinct from runtime row grouping. No icon option here (unlike the Form's next-group, which does have one). |
| `show_aggregation_in_grid` / `aggregation_summary_type` | flag / enum: `average`=0,`count`=1,`min`=2,`max`=3,`stddev`=4,`stddevp`=5,`sum`=6,`var`=7,`varp`=8 | Column footer aggregation. |
| `default_sort` / `sort_no` / `sort_order` (`asc`=0/`desc`=1) | Default row order — also the basis for runtime row grouping and an Attribute Tree's levels (§ 1). |
| `grp_until` | flag | For runtime row grouping / Attribute trees: groups every sorted column up to and including this one. |

**Verified live examples**:

- `INSIGHTS.employee` (table-level): `grid_row_height = 84` (tall enough for the `photo` column),
  `no_of_fields_locked = 2`, `grp_box_visibility = never`, `allow_edit_grid = true` with
  `grid_default_editable = false` (deliberate edit-mode, not auto-edit), `auto_resize_grid_col =
  headers_and_data`.
- `INSIGHTS.booking_hour` (aggregation): `hours_monday`…`hours_friday` and `hours_total` all carry
  `show_aggregation_in_grid = true`, `aggregation_summary_type = sum` — exactly the "hours/quantity"
  case the best-practice guidance calls out as a good aggregation candidate.
- `INSIGHTS.project`/`sub_project` (header groups): `actual_start_date`/`actual_end_date` share
  `grid_next_grp_label = actual_date`; `planned_start_date`/`planned_end_date` share
  `grid_next_grp_label = planned_date` — the canonical "Planned … | Actual …" header-group pattern.

## Step by step

1. Choose the mode up front: read-only overview, deliberately editable, default-editable
   (`grid_default_editable = true`), add-in-Grid (`allow_add_grid = true`), or analytical/aggregation
   focus — don't try to support all of them in one Grid. If the request doesn't make the mode obvious,
   ask the user rather than defaulting to a read-only overview.
2. For each column: `grid_order_no` (identity/exception → status/decision → key
   dates/quantities/owner → supporting context → audit/technical, last), `grid_type_of_col`, and
   `grid_col_width` only where the default measurement is wrong.
3. Group related column pairs under one header with `grid_field_in_next_grp`/`grid_next_grp_label`
   only when they're genuinely comparable families (Planned/Actual, Budgeted/Actual).
4. Add `show_aggregation_in_grid`/`aggregation_summary_type` only for measures with consistent
   units (hours, amount, quantity, count) — never identifiers, years, or mixed-unit columns.
5. Set `no_of_fields_locked` to the minimum identity/context needed to read later columns while
   scrolling — remember hidden columns still count toward this number.
6. Set a stable `default_sort`/`sort_no`/`sort_order` that matches the user's first decision, with a
   deterministic tie-breaker.
7. Set `auto_resize_grid_col = no` on any table/variant with 20+ visible columns.
8. Place a Grid leaf panel on the subject's screen type.
9. For a variant, run `task_setup_tab_variant_grid_overview`, then edit `tab_variant_grid`/`_overview`
   directly or seed it from the Form with `task_copy_form_to_grid`.

## Best practices and pitfalls

- **Header groups ≠ row grouping.** `grid_field_in_next_grp` decorates column *headers*;
  `default_sort`/`grp_until` plus `grp_box_visibility` drive runtime row *grouping*. Don't confuse a
  misplaced header label with a grouping bug or vice versa.
- **Aggregation scope changes with selection**: with multiple rows selected, the total is for the
  selection; with one (or zero) selected, it's for the whole dataset — this must be communicated to
  users where misreading a total is risky.
- **Over-locking** leaves no working area — lock the minimum, not everything that seems important.
- **Auto-width is expensive on wide/high-volume grids** — the documented threshold is 20+ visible
  columns; measure, don't guess.
- **Raw-key duplication**: don't show a foreign key and its already-resolved display value side by
  side without a real reason.
- **Bulk-scope ambiguity**: "All rows" tasks are capped (documented default 5,000 staged rows) and
  don't span pages for delete — never let an action's label imply broader behavior than it delivers.
- **Editable ≠ safe** — a `grid_default_editable`/auto-save Grid needs validation and concurrency
  handling designed in, not assumed.
- **Summary pitfalls**: summing mixed currencies/units without conversion; averaging averages instead
  of weighting the source values; summing snapshot balances across dates (often double-counts);
  aggregating an expensive calculated expression per row; showing so many summaries that none carries
  visual emphasis. If a total needs conversion, deduplication, weighting, or a fixed reporting cutoff,
  model a real calculated value/view/cube instead of leaning on a grid aggregation to get there.
- **Copying Grid↔Form is a starting point, not a finished design.** `task_copy_grid_to_form`/
  `task_copy_form_to_grid` are a fast way to seed one component from the other, but a Grid answers
  "which record, what state, what's comparable across rows" while a Form answers "what is this record,
  what's editable, what can wait until needed" — the same order/visibility rarely serves both jobs well
  once the copy has run. Treat the copy as a draft to then diverge from, not the final settings for
  either component.
