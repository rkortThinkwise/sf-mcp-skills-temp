---
name: thinkwise-software-factory-cubes
description: Reference guide for creating and maintaining cubes (analytical pivot/chart views) in a Thinkwise Software Factory model — the cube/cube_field/cube_field_query/cube_view/cube_view_field/cube_view_field_filter/cube_view_field_total/cube_view_field_conditional_layout/cube_view_grp family, dimensions vs. values, aggregation types, intervals and hierarchies, calculated fields, pivot and chart settings, editable values, and per-role/per-variant cube rights. Use before calling get_entity_definition/get_task_definition/execute_odata_query/stage_resource/stage_task against any of these entities, and before deciding whether a request needs a cube at all versus a Grid, report, or dedicated BI tool.
---

# Creating and Maintaining Cubes in the Thinkwise Software Factory

A **cube** is an analytical dataset built on top of one existing subject (table or view), for
interactive pivoting, slicing, and charting — not a copy of the subject's data, and not a
replacement for the subject's own Grid/Form. It classifies the subject's fields into two kinds:

- **Dimensions** — categorical fields used to segment and group: time, customer, product, region,
  status, work center.
- **Values** — measurable facts aggregated within that grouping: revenue, quantity, hours, a record
  count. (The schema's own enum literal for this is `measure`; every UI, doc, and this skill call it
  a **Value** — see "Naming traps" below.)

A **cube view** is one saved, named arrangement of those fields — which dimensions sit in which
axis, which values are shown, filters, sorting, totals, and pivot/chart presentation. One cube
commonly carries several views, each answering one specific question.

```text
Subject at a defined grain (one row = one clearly stated business fact)
  ├─ Dimensions: who / what / where / when / category
  └─ Values: amount / quantity / duration / count
        ↓
Cube view
  ├─ Filters            (restrict without becoming an axis)
  ├─ Categories/Rows     (primary drill path)
  ├─ Series/Columns      (secondary axis, low cardinality)
  └─ Values              (the aggregated numbers shown)
        ↓
Pivot table and/or chart
```

Apply this whenever an MCP connector with Software Factory access creates, inspects, or modifies a
cube, cube field, or cube view — follow the connector's standard discovery→act flow; never guess
entity/task/property names. Everything below was confirmed live against a connected model (domain
key `sf/manage_cubes` for the cube family itself; `sf/manage_datamodel` for the per-variant overrides
and screen-type wiring) — try it directly first, and only escalate to
`search_capabilities`/`get_available_domains` on a rejection rather than re-discovering a domain
that already resolved earlier this session. This skill is connector-agnostic: it names entities,
tasks, and fields, not any one connector's literal tool names.

**Companion skills, not duplicated here**: general screen-type/component mechanics (`screen_type`,
`tab`, `main_screen_type_id`, how components attach to a screen) are covered in
`thinkwise_software_factory_build_planner`'s interaction-surface step and the screen-type API-quirk
note in `thinkwise_datamodeling_guidelines` — a new screen type can't be created through the MCP
tools, so a cube's screen types (`cube`, `cube_horizontal`, `cube_no_fields` are the default
cube-capable ones) and components (Pivot table, Chart, Cube panel, Cube view bar) must be assigned to
a table by picking from what already exists. Table-variant inheritance mechanics in general (field-level vs.
snapshot, `tab_variant_change`, `tab_variant_used`) live in `thinkwise_software_factory_variants` —
this skill covers only the cube-specific override entities (`tab_variant_cube_overview`,
`tab_variant_cube_view_overview`). The general conditional-layout mechanic (colors, fonts, condition
types) lives in `thinkwise_software_factory_conditional_layouts`, but note
`cube_view_field_conditional_layout` is a **separate, cube-specific entity family**, not a row in
that skill's `conditional_layout` table — see "Conditional layouts on a cube view" below.

## Golden rule — this is a judgment-call skill, not just a mechanics reference

Building a cube field or view is a handful of API calls. The part that actually matters — and the
part every piece of Thinkwise's own documentation and community feedback stresses — is getting the
**analytical design** right before touching the model:

- **State the grain first.** What does one row of the source subject mean? ("One invoiced sales
  line," "one stock position per item and location.") If a join behind the subject can duplicate a
  business fact — history joins, many-to-many, mixing transaction rows with pre-aggregated totals —
  the cube's totals will be wrong even though every individual aggregation is technically correct.
  Validate the source subject independently before creating the cube.
- **Classify every field deliberately.** The generated proposal from `task_enrichment_create_cube`
  is a starting point, not a finished cube — the documentation explicitly calls out reviewing and
  removing semantic-free ID columns, confirming values come from the most detailed (fact-grain)
  data source, and re-checking every auto-assigned dimension/value split.
- **Classify each value's additivity.** Additive (sums cleanly across every dimension, e.g. sales
  quantity), semi-additive (sums across some dimensions but not time — e.g. an end-of-day balance),
  or non-additive (percentages, ratios, unit prices, distinct counts). A cube that silently lets a
  user sum a snapshot balance across months is easy to build and wrong to read.
- **Decide if a cube is even the right tool.** See the comparison table below — a cube is for
  repeated, rearrangeable, drill-and-pivot analysis; it is not a substitute for a Grid/Form (record
  inspection/editing), a report (pixel-perfect fixed documents), a couple of fixed KPI tiles, or an
  enterprise BI platform (Power BI/Qlik/Tableau) for cross-system governed analytics — Thinkwise
  documentation explicitly recommends OData export to those tools for that scale.
- **Ask before guessing** on: the cube's/view's/field's name when the model's naming isn't obvious,
  whether a value should be editable (see "Editable values" below — this has hard technical
  preconditions, not just a checkbox), each value's additivity classification
  (additive/semi-additive/non-additive) whenever it isn't obvious from the business meaning, any
  dimension-vs-value split that diverges from what `task_enrichment_create_cube` auto-generated, and
  whether a described "one more axis" request is actually a sign the view has become an unreadable
  everything-view (avoid one cube view with dozens of dimensions/values — prefer several focused
  views).

| Need | Prefer |
|---|---|
| Inspect or edit individual records | Grid/Form |
| A few fixed KPIs | Dashboard, tiles, or a purpose-built view |
| A pixel-perfect official document | Report |
| Cross-system, governed enterprise analytics at scale | Power BI/Qlik/Tableau via OData export |
| A fixed simple trend, no pivoting needed | A plain chart on a focused subject |
| Users repeatedly rearrange/drill/slice operational data | **Cube** |

## Object graph — confirmed live

```text
cube  (one per source subject/table, domain key sf/manage_cubes)
 ├─ cube_field            (one per dimension/value; cube_field_type: dimension | measure)
 │   └─ cube_field_query   (per-RDBMS SQL expression, for calculated/formula fields)
 ├─ cube_view              (one saved pivot/chart configuration)
 │   ├─ cube_view_field            (places one cube_field into an area of this view)
 │   │   ├─ cube_view_field_filter        (default filter values, when area = filter)
 │   │   ├─ cube_view_field_total         (extra total rows/cols, when area = value)
 │   │   └─ cube_view_field_conditional_layout  (cube-specific conditional formatting)
 │   ├─ cube_view_constant_line     (target/threshold lines on a chart axis)
 │   └─ cube_view_field_set_up      (mirror entity backing the visual "Cube set-up panel")
 ├─ cube_view_grp           (groups views into a submenu of the cube view bar)
 ├─ chart_legend_color      (custom per-member series colors on a value's chart legend)
 ├─ role_cube_overview / role_cube_field_overview   (per-role rights, incl. cube-panel & edit rights)
 └─ tab_variant_cube_overview / tab_variant_cube_view_overview   (per-table-variant overrides,
     domain key sf/manage_datamodel — see "Per-variant overrides" below)
```

Every one of these keys off `model_id, branch_id, cube_id` first, then its own id chain — the same
pattern as every other Software Factory object family.

## Creating a cube

`task_enrichment_create_cube` — bound to the `cube` entity itself, mandatory `tab_id` (the source
table), optional `cube_description`. Documented/expected to be addressable directly by the **new**
`(model_id, branch_id, cube_id)` you want to create — the same "the task creates its own bound row"
pattern used by `task_create_tab_variant` elsewhere in the model. **Verified live this did not
work**: staging this task against a not-yet-existing cube's key was consistently rejected (across
repeated attempts and domain variants) rather than creating the row. The reliable fallback, confirmed
live: skip the task entirely and create `cube` with a plain add — set `model_id`/`branch_id`/
`cube_id`/`cube_description`/`allow_dragging_fields` by hand, the same as any other top-level entity.
This forgoes the task's auto-generated field proposal, so expect to build `cube_field` rows
individually afterward (see below) — extra work for a wide table, but the only path that reliably
works.

Running the task, when it *does* work:

- Auto-generates a proposed set of `cube_field` rows (dimensions and values) from the table's columns.
- Changes the source table's screen type to a cube-capable screen type.
- Needs review, not blind acceptance — per the docs, remove semantic-free ID columns, confirm every
  value is sourced from the fact-grain table, and re-classify anything mis-detected as dimension vs.
  value.

**`cube_field` has the same fallback need.** Its declared navigation properties on `cube`
(`detail_ref_cube_cube_field_dimension`/`_value`, splitting fields by dimension vs. value) look like
the natural parent-nav path for adding one — verified live, both are rejected as invalid parent
targets for a staged add. Add `cube_field` with a plain top-level add (the full compound key supplied
directly) instead of trying to nest it under `cube`.

`cube.default_cube_view_id` sets which view opens by default; `cube.allow_dragging_fields` is the
model-level switch for the end-user "Cube panel" (drag-and-drop self-service rearranging — see
"User customization" below). `cube.olap_connection`/`olap_server_name`/`olap_db_name`/`olap_cube_name`
are legacy Windows-GUI-only OLAP cube settings (Microsoft Analysis Services) — irrelevant to a
Universal/web cube and only relevant when maintaining an old Windows application.

`task_delete_cube` removes the whole cube. `task_unlink_generated_object` (present on `cube`,
`cube_field`, `cube_view`, and more — every entity here carries `generated_by_control_proc_id`)
detaches a row that a control procedure generated, before manually editing it — the same
generated-object convention documented for scheduler/map components.

## Naming trap: "dimension"/"measure" vs. "Dimension"/"Value"

`cube_field.cube_field_type` is a two-value enum: **`dimension`** (0) and **`measure`** (1). Every
UI screen, the docs, and the 2025.1 modeler split call the second one **"Value"**, not "Measure" —
community feedback specifically flagged the old combined "Cube fields" tab as confusing before it
was split into separate **Dimensions** and **Values** tabs. When reading or writing
`cube_field_type`, use the enum literal `measure`; when talking to a user or naming things, say
"value." Don't let the schema's internal name leak into a field's user-facing description.

## Cube fields — dimensions and values

Confirmed schema (`cube_field`):

| Field | Purpose |
|---|---|
| `cube_field_type` | `dimension` or `measure` (="Value" — see above) |
| `col_id` | Source column on the underlying table |
| `summary_type` | Aggregation: `average`, `count`, `min`, `max`, `stddev`, `stddevp`, `sum`, `var`, `varp`, `formula`, `sql_expression` — only meaningful for a value |
| `summary_display_type` | How the aggregated number is shown: `default`, `percentage_of_column`, `percentage_of_row`, `percentage_of_variation`, `abs_variation` |
| `no_of_decimals` | Display precision — set by business meaning, not raw DB scale |
| `editable` | Whether this value can become an editable pivot cell (see "Editable values" — several other preconditions also apply) |
| `grp_interval` + `type_of_grp_interval` | Enables and selects an interval/bucketing for a dimension: `alphabetical`, `numeric`, `date`, `date_year`, `date_quarter`, `date_month`, `date_week_of_year`, `date_week_of_month`, `date_day_of_year`, `date_day_of_month`, `date_day_of_week`, `year_age`, `month_age`, `week_age`, `day_age` |
| `grp_interval_numeric_range` | The bucket width, for `numeric` intervals |
| `cube_field_grp_id` | **Hierarchy parent** — see below |

### Intervals

Use `type_of_grp_interval` to reduce cardinality on a raw date/number into something users can
actually scan: order date → `date_year`/`date_quarter`/`date_month`; lead time → `numeric` buckets
via `grp_interval_numeric_range`. Per the research, day-of-week numbering starts at 1 = Sunday —
confirm this matches user expectations before shipping a week-based view, and take particular care
with fiscal calendars, week numbering, and daylight-saving boundaries.

### Hierarchies — `cube_field_grp_id`

Confirmed live: `cube_field_grp_id` is a **self-referencing lookup to another `cube_field` on the
same cube**. To nest "Product" under "Product group," set the **Product** field's own
`cube_field_grp_id` to the **Product group** field's `cube_field_id` — the child row points at its
parent, the same directional pattern as a normal FK column, not the inverted pattern variants use for
tree hierarchies. Once grouped this way, the child dimension is no longer independently offered
outside its parent's hierarchy — group dimensions only when every child genuinely has one meaningful
parent throughout the analyzed period; don't group two fields together merely because they're usually
shown side by side.

**`cube_field_grp_id` alone does not make the hierarchy appear in a pivot.** Verified live: wiring the
parent-child chain via `cube_field_grp_id` and placing only the leaf field in `cube_area_category_row`
produced a flat list with no nesting at all — every ancestor level was invisible. The fix: place
**every level of the hierarchy**, not just the leaf, as its own `cube_view_field` row in the same
area, in ancestor-to-leaf order (outermost first). `cube_field_grp_id` establishes the levels'
parent-child *identity* (and is what makes a grouped child stop being independently offered in the
field picker) — it does not by itself drive nested-row rendering from a single placement. Set
`expand = true` on the outermost level's `cube_view_field` row so the tree opens one level deep by
default rather than fully collapsed.

### Calculated fields — `cube_field_query`

For `summary_type = sql_expression` (the current, non-deprecated mechanism — `formula` is the older
Windows-era equivalent), the actual expression lives in a **separate per-RDBMS row**:
`cube_field_query`, keyed by `(cube_id, cube_field_id, rdbms_type)` with a single `formula` text
field. This is how ratios and margins get computed **after** aggregation, referencing sibling cube
fields through the mandatory alias `t1`:

```sql
(t1.revenue - t1.cost) / NULLIF(t1.revenue, 0)
```

Write one `cube_field_query` row per RDBMS your branch targets (`sqlserver`, `oracle`, `postgresql`,
`iseries`). This is the correct way to build gross margin, average selling price (revenue/units),
utilization, completion percentage, and variance — always guard division by zero and null
propagation, and never average pre-computed row-level ratios when you can recompute them from
aggregated numerator/denominator instead.

## Cube views

`cube_view` is one full pivot/chart configuration. Confirmed fields, grouped by concern:

**Identity & visibility**: `cube_view_description`, `show_cube_view`, `cube_view_grp_id` +
`order_no`/`abs_order_no`, `cube_view_icon_id`, `screen_area_id`, `custom_display_type` (the same
icon/text/overflow-fallback enum used by menu items and tasks — `icon_text_*`, `text_only_*`,
`icon_only_overflow`, `overflow`, or `hidden`). Set `cube_view_icon_id` to a suitable icon as part of
creating the view, per `thinkwise_software_factory_icons` — don't leave it unset by default.

**Default presentation**: `default_cube_view_type` — `none`, `pivot_table`, or `chart`.

**Totals** (pivot): `show_col_grand_total`/`show_row_grand_total` (an extra column/row presenting
the opposite axis's grand total), `show_col_total`/`show_row_total` (subtotals), `total_position` —
`near` or `far`. Show totals only where they're mathematically meaningful — a semi-additive or
non-additive value's grand total can be actively misleading.

**Drill-down**: `drill_down_grid` — double-clicking a pivot cell opens the underlying records,
still honoring the view's filters. Essential for trust in a number, but the underlying detail must
still be covered by the subject's own row-level authorization — drill-down is not a security
boundary by itself.

**Chart settings**: `chart_type` (a large enum — 2D/3D area/bar/line/pie/doughnut/funnel/bubble/
radar/gantt/candlestick and stacked/full-stacked/side-by-side variants of most of them),
`chart_palette_id`, `chart_rotated`, `show_labels` + `label_position_bar` (`center`/`top`) +
`label_position_pie` (`inside`/`outside`/`two_columns`), `show_percentage`, `transparency` (0–100 in
steps of 10), `show_legend` + `legend_alignment_horizontal`/`legend_alignment_vertical` +
`legend_direction` + `legend_max_horizontal_percentage`/`legend_max_vertical_percentage`. Universal
maps every 3D type to its 2D equivalent for rendering and folds a few specialized/unsupported types
down to columns — design for the supported Universal behavior, since 3D rarely helps comparison
anyway.

`task_copy_cube_view`/`task_rename_cube_view`/`task_renumber_cube_view`/`task_delete_cube_view` —
same create/rename/renumber/delete family as every other ordered child object in the model.
`task_cube_view_mark_new_object_approved`/`_disapproved` — the standard new-object review workflow
(also present on `cube_field`).

### View groups — `cube_view_grp`

Groups views into a labeled submenu of the cube-view bar once there are enough views that a flat bar
becomes hard to scan (`cube_view_grp_description`, `sub_menu` flag, `icon_id`, `custom_display_type`,
`order_no`). Group by question or audience (Sales / Margin / Volume / Operations / Quality), not by
creation order. Set `icon_id` to a suitable icon representing the shared question/audience, per
`thinkwise_software_factory_icons`.

## Field placement in a view — `cube_view_field`

Places one `cube_field` into exactly one area of exactly one `cube_view`. Confirmed `cube_area` enum:

| Value | Meaning |
|---|---|
| `cube_area_filter` | Restricts the view without becoming an axis |
| `cube_area_category_row` | Primary drill path — put the broadest dimension first |
| `cube_area_series_column` | Secondary axis — keep cardinality low or the pivot gets very wide / the chart legend gets illegible |
| `cube_area_value` | The aggregated number(s) shown |
| `cube_area_menu` | A fifth value present in the schema alongside the four documented UI areas; its exact rendering isn't covered by current docs/community material — likely the "not yet placed on any axis" pool feeding the field picker. Verify in the SF UI on a live row before treating it as equivalent to one of the four areas above. |

**Setting the area**: `task_cube_view_field_area_drag_drop` is confirmed live to take **no
parameters** on either binding (`cube_view_field` or the mirror `cube_view_field_set_up`) — it exists
to back the SF's own visual drag-and-drop panel, where the target area is implied by which UI
drop-zone triggered it. Via an MCP connector, don't try to call this task — **just patch
`cube_view_field.cube_area` directly** with `stage_resource`/`patch_resource`/`commit_resource`; it's
a plain field, not a snapshot-gated one. **This field has also been seen silently dropping when set
together with other fields in one combined write** (the general "last field in a combined write can
drop" quirk — see `thinkwise_datamodeling_guidelines`) — after patching `cube_area`, re-read the row
back and confirm the value actually stuck. **Re-tested and fixed, verified live**: setting `cube_area`
*last* among the properties in one combined `stage_resource`/`patch_resource` call (after `order_no` and
any other field being changed alongside it) avoided the drop entirely — both fields held correctly with
no follow-up patch needed. Order the properties this way instead of isolating `cube_area` into its own
call.

Other confirmed fields: `order_no`/`abs_order_no` (field order within its area), `field_width`
(pixels), `sort_order` (`asc`/`desc`) + `sort_by_cube_field_id` (sort this category/series by
**another field's aggregated value** — e.g. sort customers by revenue rather than alphabetically —
distinct from simply sorting the axis's own display value), `show_top_x` + `type_of_show_top_x`
(`absolute`/`percentage`) + `show_other` (fold everything outside the Top X into an "Other" bucket —
strongly prefer enabling `show_other` whenever `show_top_x` is set, so users don't mistake a partial
ranking for the full total), `expand` (default expansion state — for a Year→Quarter→Month
hierarchy, expanding only the top level usually communicates the pattern better than expanding every
leaf).

### Default filter values — `cube_view_field_filter`

A repeatable child row per allowed/default value, keyed by `(cube_view_id, cube_field_id,
filter_value)` — only meaningful for a field placed in `cube_area_filter`. This is how a view's
default context (current company, recent years, completed-only transactions) gets modeled. Whatever
default filter you set here must remain **visible and understandable** to the user — a hidden
default filter that silently excludes canceled orders or old periods breaks reconciliation even when
the exclusion was intentional. Row-level security belongs in the subject/authorization layer, never
only in a cube view's default filter — a saved filter is a user convenience, not an access control.

### Extra totals — `cube_view_field_total`

Keyed by `(cube_field_id, summary_type)` — **a value field can carry more than one total row**, each
with its own `summary_display_type`/`no_of_decimals`/`order_no`. This is distinct from the cube
field's own base `summary_type` (used for each leaf cell): a total row can legitimately use a
*different* aggregation than the cell aggregation for a subtotal/grand-total presentation (e.g. an
average unit price at the leaf cell, but the total row instead shows a sum of the underlying
quantity via a second cube field, or the same value totaled with a different display type). Add a
total row deliberately per value, and treat a semi-additive or non-additive value's total with
extra scrutiny — a naive `sum` total row on a balance or a ratio field looks fine and is wrong.

### Conditional layouts on a cube view

`cube_view_field_conditional_layout` — a **cube-specific** conditional-layout family (not a row in
the generic `conditional_layout` entity the `thinkwise_software_factory_conditional_layouts` skill
covers). Confirmed fields: `condition_cube_field_id` (which field the condition evaluates),
`numeric_condition` (`equal_to`, `not_equal_to`, `greater_than[_or_equal_to]`,
`smaller_than[_or_equal_to]`, `between`/`not_between`, `is_empty`/`is_not_empty`), `value`/
`until_value`, and four independent apply-to flags — `apply_to_cell`, `apply_to_total_cell`,
`apply_to_custom_total_cell`, `apply_to_grand_total_cell` — plus the same
color/font/bold/italic/underline/strikethrough/font_size formatting fields as the generic mechanism,
each split into light/dark theme variants. **Test each apply-to flag independently**: a condition
that's meaningful on a leaf cell (e.g. "utilization above 95%") is frequently misleading when the
same rule also colors a grand total. This mechanism applies only to modeled standard cube views, not
ad hoc views end users build for themselves via the Cube panel.

### Chart extras — constant lines and custom series colors

- `cube_view_constant_line` — a fixed threshold/target line drawn on a chart's `x_axis` or `y_axis`
  at a given `value`, with `color`, `thickness`, `dash_style`, optional title
  (`show_title`/`title_color`/`title_font_id`/`title_alignment`), `show_in_front_or_behind` the data,
  and `show_in_legend`. Use for SLA thresholds, budget targets, or capacity limits overlaid on a
  trend.
- `chart_legend_color` — overrides the palette's automatic color for one specific value member
  (`cube_field_id` + `color_light`/`color_dark`), letting e.g. a fixed red always represent "Returns"
  regardless of palette rotation.

For making pivot values directly editable (preconditions, and the self-referencing-hierarchy
structural trap), read `references/editable_values.md` before enabling this.

## Permissions — `role_cube_overview` / `role_cube_field_overview`

Confirmed live, both role-scoped and independent of a table's own row/column rights:

| Entity | Key fields | Purpose |
|---|---|---|
| `role_cube_overview` | `available` | Whether this role can see/use the cube at all |
| | `dragging_fields_granted` (with meta-mirror `sf_allow_dragging_fields`) | Whether this role gets the end-user Cube panel (self-service rearrange), independent of whether the cube-level `cube.allow_dragging_fields` switch is even on |
| `role_cube_field_overview` | `available` | Whether this role sees this specific dimension/value at all |
| | `editable` (with meta-mirror `sf_editable`) + `granted` | Whether this role can edit this specific value — on top of every other editable-value precondition above |

Both carry `rights_icon` (`super_user`/`grant`/`read`/`hidden`/`unauthorized`) mirroring the standard
rights model used everywhere else. A cube can leak information through aggregated totals even when
individual rows stay hidden — apply and test subject/row-level authorization, cube and cube-field
availability, edit rights, drill-down access, and export permissions together, not any one of them
in isolation. Filters and hidden fields are UI conveniences, never a substitute for these rights.

## Screen types, components, and per-variant overrides

The default cube-capable screen types are `cube`, `cube_horizontal`, and `cube_no_fields`; the
available components are **Pivot table**, **Chart**, **Cube panel**, and **Cube view bar** — assigning
these to a table's screen type follows the exact same mechanics as any other screen type (pick from
what already exists in the model; creating a new screen type isn't supported via MCP — see
`thinkwise_software_factory_build_planner`). The platform removes the Pivot table component
automatically whenever no cube definition exists for the effective subject — if a screen looks like
it's missing its pivot, check that a `cube` row actually exists for that table before assuming a
component-wiring bug.

### Per-table-variant overrides — confirmed live

Same field-level inheritance pattern documented generically in `thinkwise_software_factory_variants`
(no Setup/snapshot step — plain override, tracked until it diverges):

- `tab_variant_cube_overview` (`cube_id` + `tab_variant_id`) — per-variant `default_cube_view_id` and
  `allow_dragging_fields`; `task_reset_tab_variant_cube_overview` reverts to inheriting the default.
- `tab_variant_cube_view_overview` (`cube_id` + `tab_variant_id` + `cube_view_id`) — per-variant
  `show_cube_view`, `screen_area_id`, `custom_display_type`, `conditional_layout_code`;
  `task_reset_tab_variant_cube_view_overview` reverts it.

Use these when different audiences of the same table need a different default view or a different
subset of visible views (e.g. an executive variant showing only two high-level views vs. an
analyst variant showing all of them with the Cube panel enabled) — not a reason on its own to build a
second cube.

## User customization — the Cube panel

When `cube.allow_dragging_fields` (or its per-variant override) is on and the role has
`dragging_fields_granted`, end users can rearrange fields between areas, change sort/chart settings,
and save their own personal cube views with their own filters — without touching the model. Modeled
views should still stand on their own as good starting points; a personal view can go stale when the
underlying cube fields change later, so treat this as a reason to periodically review, not a reason
to under-invest in the modeled views.

## Known SF-modeling friction (from Thinkwise Community feedback)

Worth knowing going in, since these are reported pain points rather than bugs in your modeling:

- **No live preview in the Software Factory.** Cube views can't be previewed while modeling — the
  documented workaround is building the view once in the actual running application, then
  replicating that field arrangement back into the SF. Thinkwise has stated (per a 2024 community
  reply) that a write-back/WYSIWYG editor from the running app into the SF is a longer-term direction,
  not something shipped yet — don't assume a preview or write-back capability exists.
- **Interval auto-mapping can misfire** — community reports of week/month/day interval detection
  picking the wrong granularity on the generated proposal. Always manually verify
  `type_of_grp_interval` on every interval dimension rather than trusting the generated default.
- **The 2025.1 release specifically addressed several of these complaints**: cube fields are now
  split into separate Dimensions/Values tabs (matching the `dimension`/`measure` split above), and a
  visual "Cube set-up panel" (the `cube_view_field_set_up` entity, drag-drop area assignment) was
  added. If working against an older branch/model version, some of this may not be present yet.
- Users have reported the four area names (Filters/Categories-Rows/Series-Columns/Values) not always
  matching what they expected from the running app's terminology, and that assigning a field to one
  area can visually surface elsewhere unexpectedly — when a placement looks wrong after a
  `patch_resource` on `cube_area`, re-query `cube_view_field` to confirm the stored value rather than
  trusting a stale prior read.

## Recommended workflow — new cube from scratch

1. Write down the analytical question and audience in one sentence.
2. Identify the source subject and state its exact row grain; validate independently (control
   totals) that joins behind it don't duplicate facts.
3. `task_enrichment_create_cube` with `tab_id` (+ optional `cube_description`) to generate the
   starting proposal.
4. Review every generated `cube_field`: remove semantic-free IDs, fix any mis-classified
   dimension/value, confirm `col_id` sources are fact-grain.
5. **Gate: present the proposed dimension/value classification, hierarchy plan, and view list to the
   user, and get their explicit confirmation before proceeding to any further staged writes**
   (interval/hierarchy wiring, additional views, etc.) — don't treat step 4's review as silent
   authorization to keep building. This is the `thinkwise_software_factory_mcp_base`
   "Confirm-before-mutate" convention applied at this skill's own grain.
6. Set `type_of_grp_interval`/`grp_interval_numeric_range` on date/numeric dimensions that need
   bucketing; wire `cube_field_grp_id` for genuine hierarchies.
7. For ratios/margins, add a `cube_field` with `summary_type = sql_expression` and one
   `cube_field_query` row per target RDBMS, using the `t1` alias.
8. Create one or more focused `cube_view` rows — each answering one question, not an
   everything-view.
9. Add `cube_view_field` rows placing each relevant `cube_field` into `cube_area_filter`/
   `cube_area_category_row`/`cube_area_series_column`/`cube_area_value` — patch `cube_area` directly
   rather than via the drag-drop task.
10. Configure `cube_view_field_filter` defaults, `sort_order`/`sort_by_cube_field_id`,
    `show_top_x`/`show_other`, and `expand` per field.
11. Configure `cube_view` totals (`show_*_total`, `total_position`), `drill_down_grid`, and
    `default_cube_view_type` (pivot vs. chart).
12. For a chart view: `chart_type`, `chart_palette_id`, labels/legend, and any
    `cube_view_constant_line`/`chart_legend_color` overrides.
13. Add `cube_view_field_conditional_layout` sparingly, testing each apply-to-cell/total/grand-total
    flag separately.
14. Decide `cube.allow_dragging_fields` and per-role `dragging_fields_granted`; grant
    `role_cube_overview`/`role_cube_field_overview` rights.
15. Only enable `editable` on a value after confirming every precondition in "Editable pivot values"
    holds for its intended grain.
16. Wire the table's screen type/components — picking an existing cube-capable screen type, since
    creating a new one isn't supported via MCP (see `thinkwise_software_factory_build_planner`) — and
    any per-variant overrides (`tab_variant_cube_overview`/`tab_variant_cube_view_overview`).
17. Validate totals against an independently calculated control total, check performance at
    production-like cardinality, and test light/dark conditional-layout themes.

## Practical example — a sales revenue cube

Source subject: `sales_order_line`, grain = one invoiced order line.

- Dimensions: `order_date` (interval `date_month`, hierarchy: `order_date_year` as
  `cube_field_grp_id` parent of `order_date_quarter`, which is in turn the parent of
  `order_date_month`), `customer_id`, `product_id` grouped under `product_group_id`
  (`cube_field_grp_id` on the product field points at the product-group field), `region_id`.
- Values: `revenue` (`summary_type = sum`), `quantity` (`sum`), `unit_price`
  (`summary_type = sql_expression`, `cube_field_query.formula = t1.revenue / NULLIF(t1.quantity, 0)`
  — never a plain `average` of the stored unit price column, which would misweight lines of
  different quantity).
- Views:
  - **Revenue trend** — `order_date_month` in `cube_area_category_row`, `revenue` in
    `cube_area_value`, `default_cube_view_type = chart`, `chart_type = 2d_line`.
  - **Revenue by customer and product** — `customer_id` in `cube_area_category_row`,
    `product_group_id` in `cube_area_series_column`, `revenue`+`quantity` in `cube_area_value`,
    `show_top_x = 10` with `show_other = true` sorted by `revenue` (`sort_by_cube_field_id`), pivot
    presentation with `show_row_total`/`show_col_grand_total` on.
- Conditional layout: `condition_cube_field_id = revenue`, `numeric_condition = smaller_than`,
  `value = 0`, `apply_to_cell = true`, red background — flags a reversed/negative order line at a
  glance.
- Rights: `role_cube_overview.available = true` for Sales role; `dragging_fields_granted = true`
  only for the Analyst role; `role_cube_field_overview.editable` left `false` everywhere — this is a
  historical-analysis cube, not a planning input.

## Pre-flight checklist

- **State the grain and validate control totals before creating anything** — a technically correct
  `sum` over duplicated rows is still a wrong business answer.
- **`task_enrichment_create_cube` can reject a not-yet-existing cube's key outright** — verified live;
  don't spend more than one retry on it. Fall back to a plain add on `cube` (and, separately, on
  `cube_field` — its declared dimension/value navs from `cube` aren't valid parent-nav targets
  either).
- **Review the generated proposal — never accept it unchanged.** Remove ID-only fields, confirm
  fact-grain sourcing, re-check every dimension/value classification.
- **`cube_field_type` enum literal is `measure`; call it "Value" everywhere user-facing** — don't
  let the schema name leak into descriptions or conversation.
- **`cube_field_grp_id` is a self-referencing parent pointer** — set it on the *child* dimension to
  build a hierarchy; this is the opposite direction of variant-tree `parent_col_id` documented in
  the variants skill, so don't assume the two work the same way.
- **Wiring `cube_field_grp_id` is not enough to see a hierarchy in the pivot** — every level, not
  just the leaf, needs its own `cube_view_field` placement in the same area (ancestor-to-leaf order),
  or the tree renders flat with no nesting at all.
- **A self-referencing hierarchy (org chart, BOM, category tree) sharing its table with an editable
  value is a structural trap**, not just a permissions edge case — a parent with exactly one child can
  look editable and silently write into the child's row instead. Give the hierarchical/browsing view a
  second, `editable = false` field on the same column; keep the real editable field only in a
  separate flat view.
- **Calculated/ratio fields live in `cube_field_query`, one row per RDBMS, alias `t1`** — never as a
  plain average of a stored ratio column.
- **Set `cube_area` on `cube_view_field` directly via `patch_resource`** — the
  `task_cube_view_field_area_drag_drop` task carries no parameters and exists for the SF's own
  visual panel, not for programmatic placement.
- **`show_top_x` should almost always pair with `show_other = true`** — a partial ranking without
  "Other" looks like the complete picture.
- **An editable value needs all six preconditions at once** (permission + is-a-value + column
  editable + aggregation in {min,max,sum,average} + exactly-one-row-per-cell + the per-role
  `editable` grant) — missing any one silently keeps the cell read-only or produces an ambiguous
  write-back.
- **`cube_view_field_conditional_layout` is a separate entity family from the generic
  `conditional_layout` table** — don't look for cube formatting rules there.
- **Check `role_cube_overview`/`role_cube_field_overview` independently of subject rights** — a cube
  can expose an aggregate over rows a role couldn't individually see; totals are not automatically
  safe just because row-level security exists elsewhere.
- **No live preview exists in the SF for a cube view** — expect to verify the real arrangement in
  the running application, especially on an older branch predating the 2025.1 set-up panel.
