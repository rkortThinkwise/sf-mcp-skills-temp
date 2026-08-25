---
name: thinkwise-software-factory-conditional-layouts
description: Reference guide for conditional layouts (conditional formatting) in a Thinkwise Software Factory model — data-driven styling such as status colours on tables, tasks, and cube views, including row/column targeting, light/dark themes, and accessibility. Use before inspecting or modifying any conditional-layout entity via an MCP connector with Software Factory access, or whenever asked to add or change status colours, warning highlights, or any "make this red/bold/green" styling.
---

# Conditional Layouts (Conditional Formatting) in the Thinkwise Software Factory

Thinkwise calls this feature **conditional layout**; despite the name, its purpose is conditional
*formatting* — changing how a value or row *looks* based on data, never what the user is allowed to do
with it. A scan of 71 model exports found 4,058 standard conditional-layout definitions (~34,000
condition records), dominated by status visualization (1,467 definitions with `status` in their
identifier) — followed by completion, progress, blocked records, invalid/missing data, priority,
changed/deleted records, and planning states.

Apply this skill whenever an MCP connector with Software Factory access is used to create, inspect, or
troubleshoot conditional layout — follow the connector's standard discovery→act flow; never guess
entity/task/property names. This skill is connector-agnostic: it names entities, fields, and enums, not
any one connector's literal tool names.

## Verified domain map

Every entity below was confirmed live against a real connected model. Conditional layout is **not one
entity family** — it repeats, independently, once per object type it can decorate, in whichever domain
that object type otherwise lives:

| Object type | Entities | Domain (confirmed) |
|---|---|---|
| Table (grid/form/edit/scheduler resource) | `conditional_layout`, `conditional_layout_condition`, `conditional_layout_tag` | `manage_datamodel`-style — note there's no Card list or Tree surface among the `apply_to_*` flags below; see `thinkwise_software_factory_subject_components` for those two components generally |
| Task parameter form | `task_conditional_layout`, `task_conditional_layout_condition`, `task_conditional_layout_tag`, `task_variant_task_conditional_layout` (per-variant override) | `manage_tasks`-style |
| Cube view (cells/totals) | `cube_view_field_conditional_layout` (condition fields are inline — no separate condition child, confirmed) | `manage_cubes`-style |
| Scheduler time cells | `scheduler_view_conditional_layout`/`_condition`/`_tag` | `manage_scheduler`-style — see `thinkwise_software_factory_scheduler_component`, not duplicated here |
| Report parameter form | `report_conditional_layout`, `report_conditional_layout_condition` | **Not reachable through this connector** — see "Known gap: report conditional layout" below |

Try the domain implied by the object type directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session.

## What conditional formatting does and doesn't do

It can change: background colour, font colour, bold/italic/underline/strikethrough, font size (L/XL),
one cell or the entire row, grid/form/edit/scheduler-resource presentation, task/report parameter
styling, and cube cells/totals.

It cannot: prevent invalid data, make a field mandatory or read-only, hide a task, change
authorization, execute business logic, or guarantee a user acts on a warning. Those need validation,
layout logic, context logic, rights, or database logic instead.

> Conditional formatting communicates a state; it should not be the only mechanism enforcing that state.

| Requirement | Use instead |
|---|---|
| Show a late delivery date in red | Conditional layout (this skill) |
| Make a delivery date mandatory | Layout procedure |
| Disable a task for completed orders | Context procedure |
| Prevent an invalid delivery date | Default, task, handler, or constraint |
| Show the number of late orders | Badge |
| Only show late orders | Prefilter (`thinkwise_software_factory_prefilters`) |
| Explain why a value is red | Conditional-layout help text, or a supporting status/message column |

## Before creating anything

Follow `thinkwise_software_factory_mcp_base`'s "Shared conventions" section (confirm-before-mutate,
ask-don't-default) at this skill's own grain: before the first `stage_resource`/`commit_resource` call,
propose to the user — in plain language — the target column(s) or rows to style, the condition(s) that
will trigger each, and the colour/severity mapping for each state. Use the "Common use cases" colour
table below as a starting point to *propose*, not a default to apply silently. Get explicit confirmation
before staging anything, particularly when the target column is one of several plausible choices (see
"Row vs. column targeting") or a state's severity is genuinely ambiguous (see "Common use cases").

## Table-level `conditional_layout` — field reference (verified)

Keyed by `(model_id, branch_id, tab_id, conditional_layout_id)`.

| Field | Notes |
|---|---|
| `conditional_layout_description` | Translatable label |
| `show_conditional_layout` | **Master on/off switch** — verified live: several real rows carry a fully-configured colour/condition with `show_conditional_layout = false`. A layout can be completely modeled and left dormant; don't assume every row you find is actually active. |
| `col_id` | Target column. **Blank = whole row** (see "Row vs. column targeting" below) |
| `apply_to_grid` / `apply_to_form` / `apply_to_edit` / `apply_to_scheduler_resource` | Independent boolean flags — **not** a single enum. (A synthetic example with one combined `apply_conditional_layout` field is not the real shape; ignore any doc that implies a single field here.) |
| `font_id` | Legacy Windows-GUI-only named font — verified as a thin `(font_id, font_info)` lookup, superseded by `font_size` (L/XL) for Universal UI. Leave unset for new Universal UI work. |
| `background_color` | Legacy single Windows-GUI-only background colour (`Edm.Int32`). No `font_color` legacy equivalent exists — only background has this legacy/theme-pair asymmetry. |
| `background_color_light` / `background_color_dark` | **Universal UI** per-theme background colour — set both |
| `font_color_light` / `font_color_dark` | **Universal UI** per-theme font colour — set both |
| `bold` / `italic` / `underline` / `strikethrough` | Booleans |
| `font_size` | Enum: `L` = `0`, `XL` = `1`. Absent = default size |
| `generated_by_control_proc_id` | Set only if a control procedure owns/regenerates this row |

**No `order_no`/priority field exists on `conditional_layout`** — verified against the full live property
list. Unlike `tab_prefilter` (which has `order_no`), there is no administrator-settable evaluation
priority for overlapping table conditional layouts. This reinforces, with a concrete mechanism, why the
"make conditions mutually exclusive" guidance below matters more here than it might first appear: there
is no priority number to fall back on if two layouts both match the same row. (Scheduler time-cell
layouts are the exception — `scheduler_view_conditional_layout` is keyed with an explicit
`cell_color_no`; that family also documents "first match wins" — see the scheduler skill.)

Bound tasks (verified): `task_copy_conditional_layout` (`from_tab_id`/`from_conditional_layout_id`/
`to_tab_id`/`to_conditional_layout_id`), `task_delete_conditional_layout`,
`task_rename_conditional_layout`, `task_show_history`, `task_unlink_generated_object`.

**Creating this row, verified**: a plain/unscoped add of a new `conditional_layout` record is rejected
as a weak/dependent-entity error, with candidate parents `tab` or `model_settings_modeler` — the same
shape of rejection documented for `scheduler` in `thinkwise_software_factory_scheduler_component`. Add
it as a detail of its owning `tab` record (supplying `model_id`/`branch_id`/`tab_id` as the parent key)
rather than as a standalone create.

### `conditional_layout_condition` — field reference (verified)

Keyed by `(model_id, branch_id, tab_id, conditional_layout_id, conditional_layout_condition_no)`. A
layout's conditions are AND-ed together; a layout with zero condition rows is always applied
(`show_conditional_layout` permitting). `conditional_layout_condition_no` is typically presented as a
non-editable/system-assigned field when adding a new condition through a staged create flow — the
platform assigns its actual value on commit rather than the caller supplying one; don't try to compute
or guess a sequential number for it.

| Field | Notes |
|---|---|
| `col_id` | The column this **condition** evaluates — independent of the parent's own `col_id` (the column that gets *styled*). Verified live: `booking_hour.friday_total_hours_greater_than_8` styles `hours_friday` but conditions on `total_hours_friday` — the coloured column and the evaluated column are routinely different. |
| `condition` | 18-value enum (verified, identical set to `tab_prefilter`'s filter condition) — see table below |
| `type_of_value` | `constant` = `0` · `column` = `1` |
| `value` | Constant literal, when `type_of_value = constant` |
| `value_col_id` | Comparison column, when `type_of_value = column` |
| `until_type_of_value` / `until_value` / `until_value_col_id` | Same constant/column choice, for the second bound of `between`/`not_between` |

Verified condition enum (all 18 values, live):

| Value | Condition | | Value | Condition |
|---|---|---|---|---|
| 0 | Equal to | | 9 | Does not contain |
| 1 | Not equal to | | 10 | Is empty |
| 2 | Greater than | | 11 | Is not empty |
| 3 | Smaller than | | 12 | Does not start with |
| 4 | Greater than or equal to | | 13 | Not between |
| 5 | Smaller than or equal to | | 14 | Ends with |
| 6 | Between | | 15 | Does not end with |
| 7 | Starts with | | 16 | In |
| 8 | Contains | | 17 | Not in |

**Not independently verified**: the exact serialization `value` needs for `in`/`not_in` (e.g. a
delimited list) — test against a live add before relying on a specific separator.

Real verified example (`booking_hour.friday_total_hours_greater_than_8`): one condition row,
`col_id = total_hours_friday`, `condition = 2` (greater_than), `type_of_value = 0` (constant),
`value = "8"`. A believable corrected shape of the doc's illustrative JSON:

```json
{
  "tab_id": "purchase_order",
  "conditional_layout_id": "order_status_completed",
  "show_conditional_layout": true,
  "col_id": "order_status",
  "apply_to_grid": true,
  "apply_to_form": true,
  "apply_to_edit": true,
  "apply_to_scheduler_resource": false,
  "background_color_light": -4684277,
  "background_color_dark": -4684277,
  "bold": false,
  "italic": false
}
```
```json
{
  "tab_id": "purchase_order",
  "conditional_layout_id": "order_status_completed",
  "conditional_layout_condition_no": 12345,
  "col_id": "order_status",
  "condition": 0,
  "type_of_value": 0,
  "value": "COMPLETED"
}
```
The numeric colour/enum values represent platform enumerations — maintain these through the Software
Factory's own named settings rather than hand-editing the integers.

### `conditional_layout_tag`

Plain `(tag_id, value)` pairs per layout — the same generic tagging mechanism used elsewhere in the
model (arbitrary metadata, not styling). Bound tasks: `task_show_history`,
`task_unlink_generated_object` only (tags are added/edited directly, not copy/rename/deleted as a unit).

## Row vs. column targeting

Leave `col_id` **empty** to colour the entire row; set it to colour one cell. The scan's own ratio —
383 row-level layouts vs. 3,675 column-level, plus 440 with no condition at all (always-applied) — shows
a strong platform-wide preference for focused cell formatting.

**Default to a specific column.** It gives the user a direct visual link between the problem and the
value, and stacks more legibly with other row content. Reserve whole-row formatting for genuinely
record-level states: blocked, cancelled/deleted, an entire failed message, an unavailable production
order, an urgent safety condition. Avoid whole-row for one missing field, an ordinary status column, or
several independent conditions layered onto one row — those read more clearly as separate,
column-targeted layouts. When more than one column is a plausible target, ask the user which one rather
than picking unilaterally.

## Where to apply it: `apply_to_grid` / `apply_to_form` / `apply_to_edit` / `apply_to_scheduler_resource`

Four independent booleans, verified as such (not a combined enum):

- **`apply_to_grid`** — comparative scanning across many rows: statuses, deadlines, blocked orders,
  stock shortages, integration failures, planning conflicts. The most natural home for most layouts.
- **`apply_to_form`** — state that stays relevant while viewing one record: invalid/incomplete fields,
  important record state, values needing attention before an operation. Less useful for comparative
  facts ("highest value in this list").
- **`apply_to_edit`** — only when the formatting actively helps *while editing*: a value that becomes
  invalid after another field changes, a threshold crossed mid-edit. Verified live: every real row
  sampled had `apply_to_edit = true` set **together with** `apply_to_grid` and `apply_to_form` — this
  matches the platform requirement that Apply to Edit be combined with Grid and Form, not used alone.
  Don't enable it if it would obscure the normal mandatory-field indicator or look like a hard
  validation error while editing is still permitted. **`apply_to_edit` cannot be set directly through a
  staged write** — verified live it comes back read-only/system-derived from `apply_to_grid` and
  `apply_to_form`'s own values, not an independently settable flag; don't try to patch it, set the other
  two instead and let it follow.
- **`apply_to_scheduler_resource`** — colours the **resource** row/label in a Scheduler's grouping
  panel rather than an activity bar (grey out an unavailable resource, tint hierarchy levels). See
  `thinkwise_software_factory_scheduler_component` for the Scheduler-specific evaluation rule
  ("first matching record per resource wins") and the HTML-formatting interaction (HTML/Multiline
  title/tooltip columns silence conditional layout, specifically font-size/strikethrough/underline).
- **No `apply_to_card_list` or `apply_to_tree`** — confirmed this is the complete flag set, so neither
  component has its own dedicated conditional-layout surface. Whether either inherits `apply_to_grid`'s
  or `apply_to_form`'s styling at render time hasn't been independently confirmed — see
  `thinkwise_software_factory_subject_components`'s Card List and Tree sections for the same open
  question from that side, and test empirically before relying on it.

Universal UI does not support conditional layouts on radio-button, signature, checkbox, or HTML
controls, per platform documentation.

## Colour and font fields — legacy vs. Universal UI

Every layout family carries a **legacy, single-value Windows-GUI field** alongside a **Universal-UI
theme pair**, confirmed on `conditional_layout`/`task_conditional_layout`:

| Concept | Legacy (Windows GUI) | Universal UI (set both) |
|---|---|---|
| Background colour | `background_color` | `background_color_light` + `background_color_dark` |
| Font colour | *(no legacy equivalent — asymmetric)* | `font_color_light` + `font_color_dark` |
| Font family/size | `font_id` → `font` lookup (`font_info` string only) | `font_size` enum (`L`=0, `XL`=1) |

For new work targeting Universal UI, ignore `background_color`/`font_id` and always set both the light
and dark variant of whichever colour you're using. Don't copy one theme's colour into the other without
checking contrast — a light-yellow background with dark text can disappear in dark mode; a saturated
red background can overwhelm; grey "inactive" text can become unreadable. Test normal, selected,
focused, hover, and edit states, with real data density.

### Computing a colour value without the Software Factory's own colour picker

`background_color_light`/`_dark` and `font_color_light`/`_dark` store a plain signed `Edm.Int32` with
no documented encoding. Verified live (on `scheduler_view_conditional_layout.background_color_light`/
`_dark`, a sibling field with the identical shape): assuming full opacity, the stored value is

```
signed_int32 = (R * 65536 + G * 256 + B) - 16777216
```

where `R`/`G`/`B` are each 0–255 from the desired colour's hex value. Confirmed against a real example:
`-7223041` decodes to `0x91C8FF` (a light blue), matching a cell colour named for that shade.

**Prefer setting the colour through the Software Factory's own colour setting/picker when that UI is
reachable** — this formula is the fallback for a connector/API-only session with no UI access, not a
replacement for the platform's own colour management.

## Choosing conditions

Same reasoning as prefilters/tasks (see `thinkwise_software_factory_prefilters` for the general
column-vs-query framing), applied to the 18-value enum above:

- **Equal to / Not equal to** — discrete states (`status = BLOCKED`, `interface_status != PROCESSED`).
  With Not equal to, decide deliberately whether `NULL` should count as exceptional. For a
  boolean-domain column, the plain string literal `"true"`/`"false"` is accepted directly as the
  constant `value` — no special encoding needed.
- **Greater/smaller than, Between** — thresholds and bands (`stock_quantity < minimum_stock`,
  `progress between 80 and 99`). Use `type_of_value = column` (`value_col_id`) to compare two columns on
  the same row instead of hardcoding a threshold — more maintainable than duplicating a constant across
  many layouts.
- **Contains/starts with/ends with** — sparingly, for structured codes/prefixes/filenames. Don't derive
  business state by searching free text a user typed; add a real status/expression column instead.
- **Is empty / Is not empty** — missing-but-important data (`external_reference is empty`). If the value
  is actually mandatory, back it with real validation too, not just the colour.
- **In / Not in** — verified to exist in the enum; format of `value` not independently confirmed, test
  before relying on it.

**Expression fields** are the platform's answer to conditions needing several business rules, date
arithmetic, aggregation, related-table data, complex `NULL` handling, or role-dependent behaviour —
model a `case`-based expression column (`is_overdue`, `has_shortage`, `requires_approval`) and condition
the layout on a simple equality against it, keeping the business logic in one place instead of
duplicated across many condition rows.

## Task-level `task_conditional_layout` — field reference (verified)

Keyed by `(model_id, branch_id, task_id, conditional_layout_id)`. Tasks and reports have no grid, so
their conditional layouts style **parameters on the input form** instead of a column: `col_id` is
replaced by `task_parmtr_id`, and there is no `apply_to_*` flag family (a task form has only the one
surface). Otherwise the same styling fields as the table family
(`background_color_light`/`_dark`, `font_color_light`/`_dark`, `bold`/`italic`/`underline`/
`strikethrough`, `font_size`, legacy `background_color`/`font_id`).

`task_conditional_layout_condition` mirrors `conditional_layout_condition` with parameter references in
place of column references: `task_parmtr_id` (condition's own evaluated parameter), `value_task_parmtr_id`
/ `until_value_task_parmtr_id` (for `type_of_value = column`-equivalent comparisons against another
parameter). Same 18-value `condition` enum, same `constant`/`column`-style `type_of_value` enum.

**`task_variant_task_conditional_layout`** — keyed by `(..., task_id, task_variant_id,
conditional_layout_id)`, exposing only `show_conditional_layout`: a per-variant on/off override of a
layout defined at the base task, the task equivalent of `tab_variant_prefilter_overview`'s per-variant
state override. Tables have **no equivalent per-variant override** for `conditional_layout` — verified
absent (see "Known gap" below) — table conditional layouts are table-wide only.

Good task/report use cases (styling, not validation, which must still be added separately): a required
parameter needing attention, a selected quantity exceeding available stock, a chosen date outside the
planning window, a destructive option that's been selected, a missing external-system identifier.

Bound tasks mirror the table family: `task_copy_task_conditional_layout`
(`from_task_id`/`from_conditional_layout_id`/`to_task_id`/`to_conditional_layout_id`),
`task_delete_task_conditional_layout`, `task_rename_task_conditional_layout`, `task_show_history`,
`task_unlink_generated_object`.

## Cube-level `cube_view_field_conditional_layout` — field reference (verified)

Keyed by `(model_id, branch_id, cube_id, cube_view_id, cube_field_id, conditional_layout_id)`. Cube-view
conditional layouts are a distinct, flat sub-family (condition inline, no child entity) with their own
gradient/multi-surface fields — see
`references/cube_level_conditional_layout.md` for the full verified field reference before creating or
inspecting one of these rows.

## Known gap: report conditional layout

`report_conditional_layout`, `report_conditional_layout_condition`, and
`report_variant_report_conditional_layout` all exist as real, distinct translatable object types in the
model (confirmed via the live `transl_object.type_of_object` enum — see "Translation" below). None were
reachable as entity sets through this connector, in any of the domains checked
(`manage_datamodel`, `manage_control_procedures`, `manage_screentypes`, `manage_tasks`) — even the
plain `report` entity exposed here is a bare `(model_id, branch_id, report_id)` stub with no navigation
to parameters or conditional layouts at all. Only `report_conditional_layout_tag` (the child tag table)
was reachable, orphaned, with no way to reach its own parent through this connector.

- **Check the connector's own domain metadata first** — a different or newer connector/domain may expose
  reports more fully than the one checked here.
- **If it doesn't**: report conditional layouts likely need the Software Factory's own UI directly for
  now. Don't assume the feature doesn't exist in the platform — only that this connector's exposed
  surface doesn't reach it. The field shape is presumably parallel to `task_conditional_layout`
  (`report_parmtr_id` in place of `col_id`/`task_parmtr_id`) but this was **not independently verified**
  given the access gap — confirm against the Software Factory UI or a fuller connector before assuming
  exact field names.

Similarly, **`tab_variant_conditional_layout`** (a per-table-variant override, analogous to
`task_variant_task_conditional_layout`) exists as a translatable object type but was not found as a
reachable entity set in `manage_screentypes` or `manage_datamodel` — table-level conditional layouts
appear to be genuinely table-wide only through this connector, with no per-variant override surface
found. Re-check if a variant-scoped override is specifically needed.

## Related but distinct: chart legend colour

`chart_legend_color`/`chart_legend_color_condition` are a **separate** mechanism (confirmed as distinct
translatable object types) for colouring chart legend entries/series — not part of the
`conditional_layout` family and not covered by this skill. Don't conflate a request to "colour a chart
series by status" with a table/task/cube conditional layout; it's a different entity family under the
chart configuration itself.

## Translation

Every layout name is a translatable object, generated with a bracket-placeholder default like any other
model object — see `thinkwise_software_factory_translation_objects` for the full mechanics. Confirmed
live `type_of_object` values (from the model's own enum, not guessed):

| Object | `type_of_object` |
|---|---|
| `conditional_layout` | 44 |
| `conditional_layout_condition` | 45 |
| `conditional_layout_tag` | 573 |
| `task_conditional_layout` | 590 |
| `task_conditional_layout_condition` | 591 |
| `task_conditional_layout_tag` | 593 |
| `task_variant_task_conditional_layout` | 594 |
| `report_conditional_layout` | 585 |
| `report_conditional_layout_condition` | 586 |
| `report_conditional_layout_tag` | 588 |
| `report_variant_report_conditional_layout` | 589 |
| `cube_view_field_conditional_layout` | 49 |
| `cube_field_conditional_layout` (legacy) | 225 |
| `tab_variant_conditional_layout` | 283 |
| `scheduler_view_conditional_layout` / `_condition` / `_tag` | 1032 / 1033 / 1034 |
| `chart_legend_color` / `_condition` | 1037 / 1038 |

Before considering a new layout done, check `transl_object_transl` across every language the branch
supports (`branch_appl_lang`) for lingering `[bracket]`-placeholder text, rather than assuming one write
covered every configured language.

## Common use cases (design guidance)

Design guidance on *what* to condition on and *how* to style it (status-colour table, missing-data,
planning, deadline, financial, and generated-value patterns) — see
`references/design_patterns.md` before proposing a colour/severity mapping to the user.

## Overlapping layouts

Remember: no `order_no`/priority field exists here — see above. Make conditions mutually exclusive
rather than relying on evaluation order:

```text
Problematic — one record can match all three, outcome unpredictable:
  Layout A: due_date < today
  Layout B: status != completed
  Layout C: priority = urgent

Better — mutually exclusive:
  critical_overdue:  due_date < today AND status != completed AND priority = urgent
  normal_overdue:    due_date < today AND status != completed AND priority != urgent
```

Or build one expression field representing the presentation state (`attention_level` =
`CRITICAL`/`WARNING`/`NORMAL`) and define one mutually-exclusive layout per value.

## Accessibility and usability

Never communicate meaning by colour alone — reinforce with at least one of: status text, an icon or
domain element, a badge, a warning message, help text, a dedicated boolean/attention column, or a font
treatment in addition to background colour. Reserve red for actual problems, avoid red/green as the only
distinction, keep text contrast high in both themes, avoid colouring most rows on a screen, and confirm
selected/focused rows stay legible. Conditional-layout help text can be surfaced in the generated
subject help — useful for explaining a colour legend to end users.

## When not to use conditional layout

- The user must be prevented from doing something → validation/rights, not formatting.
- Records should be removed from the screen entirely → a prefilter.
- A field should become hidden/mandatory/read-only → layout logic.
- A task must be unavailable → context procedure/rights.
- A warning must be acknowledged, or the rule must also hold through the API → real validation.
- Nearly every record would be highlighted, or the screen is already visually dense → reconsider scope.
- Universal UI does not support it on radio-button, signature, checkbox, or HTML controls.

## Pre-flight checklist

- Confirm the connector's actual domain key before assuming a fixed name holds: table layouts under
  `manage_datamodel`-style, task layouts under `manage_tasks`-style, cube layouts under
  `manage_cubes`-style — check `manage_scheduler`/`manage_datamodel` for the scheduler-cell family.
- **Default to a specific `col_id`/`condition_cube_field_id`/`task_parmtr_id` target**, not a blank
  whole-row layout — the platform's own usage is ~90% column-level.
- **Set `show_conditional_layout` deliberately** — a fully-configured layout can exist with it `false`;
  don't assume every row you find is active, and don't forget to flip it on when you actually want the
  layout live.
- **Set both light and dark variants of every colour** (`background_color_light`/`_dark`,
  `font_color_light`/`_dark`) — the legacy single-value fields (`background_color`, `font_id`) are
  Windows-GUI-only and ignored by Universal UI.
- `apply_to_edit` should be combined with `apply_to_grid` and `apply_to_form`, not enabled alone. It's
  read-only/derived, not independently settable — set `apply_to_grid`/`apply_to_form` instead.
- **When targeting the scheduler-resource surface** (`apply_to_scheduler_resource = true` with
  `apply_to_grid`/`apply_to_form = false`), set `apply_to_scheduler_resource` to `true` *before* setting
  `apply_to_grid`/`apply_to_form` to `false` — verified live, doing it in the other order gets
  `apply_to_grid` silently reset back to `true` by the layout engine.
- The condition's own evaluated column/parameter/field (`conditional_layout_condition.col_id`,
  `task_conditional_layout_condition.task_parmtr_id`, `cube_view_field_conditional_layout.
  condition_cube_field_id`) can legitimately differ from the parent row's *target* — don't assume they
  must match.
- Use `type_of_value = column` (`value_col_id`/`value_task_parmtr_id`) for a threshold that varies by
  record, instead of hardcoding a constant across many layouts.
- Remember: layouts have no `order_no`/priority field — see above. Make overlapping conditions mutually
  exclusive, or drive them off one expression field.
- `cube_view_field_conditional_layout` is flat (condition inline, no child entity) and supports a
  two-colour gradient plus per-surface flags (`apply_to_cell`/`_total_cell`/`_custom_total_cell`/
  `_grand_total_cell`) and an attached image — richer, and structurally different, from the table/task
  families.
- **Report conditional layouts are a known access gap through this connector** — the entities exist in
  the model (confirmed via `transl_object`) but aren't reachable here; don't spend time hunting for a
  `report_conditional_layout` entity set before checking whether a different connector/domain exposes
  it, and say so explicitly if it doesn't.
- **Table-level conditional layouts have no per-variant override** — unlike tasks
  (`task_variant_task_conditional_layout`) and prefilters (`tab_variant_prefilter_overview`), no
  reachable `tab_variant_conditional_layout`-style entity was found; treat a table's conditional layouts
  as table-wide.
- Conditional layout is translatable like any other object — check every configured branch language for
  lingering bracket-placeholder text before calling a new layout done.
- Don't conflate `chart_legend_color`/`_condition` (chart series colouring) with this family — it's a
  separate mechanism.
