---
name: thinkwise-software-factory-scheduler-component
description: Reference guide for creating and maintaining a Scheduler component in a Thinkwise Software Factory model — the scheduler, scheduler_view, scheduler_view_resource_col, and scheduler_view_conditional_layout(_condition/_tag) entities. Use before calling get_entity_definition/execute_task/execute_odata_query against scheduler-related entities (including tab_task/task/process_flow/process_action for a scheduler-bearing table), and before writing or reviewing the control procedure template behind a scheduler view or its update handler.
---

# Setting Up and Maintaining a Scheduler Component in the Thinkwise Software Factory

Reference for the Scheduler's full lifecycle: the underlying subject's data model (one non-nullable
primary key, resource + activity rows in a single view) → `scheduler` (table-level config: resource
grouping, activity-linked columns, drag permissions) → one or more `scheduler_view` rows (timescales,
pagination, time-cell display) → `scheduler_view_resource_col` (extra read-only columns in the
resource panel) → `scheduler_view_conditional_layout`/`_condition`/`_tag` (time-cell colouring) →
screen (a dedicated Scheduler-only screen type) → tasks and process flows (click-to-create,
double-click-to-detail, external drag-and-drop, jump-to-date). Every entity/field/value below was
confirmed live against a real model (`sf/manage_scheduler` domain, with some of the same entities also
reachable via a `sf/manage_datamodel`-style domain in at least one connector) and against a working
reference application (`PROJECT_MANAGER`) implementing multi-table hierarchical resource planning with
HTML-formatted activities — not guessed from documentation.

Apply this whenever an MCP connector with Software Factory access (`sf_mcp`, `indicium`) is used to
create, inspect, or troubleshoot a Scheduler component — follow the connector's standard
discovery→act flow; never guess entity/task/property names.
`scheduler`/`scheduler_view`/`scheduler_view_resource_col`/`scheduler_view_conditional_layout*`
typically live in a `manage_scheduler`-style domain; the underlying subject's `tab`/`col`/`ref` live in
a `manage_datamodel`-style domain (which may *also* expose `scheduler` and
`scheduler_view_conditional_layout*` directly — check both if the first search comes back empty rather
than assuming a single domain owns them); `control_proc`/`control_proc_template` (the view's SQL,
update handlers) live in a `manage_control_procedures`-style domain; `tab_task`/`task` live in a
`manage_tasks`-style domain; `process_flow`/`process_action` live in a `manage_process_flows`-style
domain — try these directly first, and only escalate to `search_capabilities`/`get_available_domains`
on an `entity_set_not_found`/`domain_not_found`-style rejection rather than re-discovering domains that
already resolved earlier this session.

For general data-modeling rules (naming, domain reuse, reference direction, unique indexes) see
`thinkwise_datamodeling_guidelines`. For how to actually write and generate the SQL behind a scheduler
subject that's a view (`tab` → `control_proc`/`control_proc_template` →
`template_prog_object_item` → generated `CREATE VIEW`, including the two-step "generate code group"
then "generate object code" sequence) see `thinkwise_software_factory_create_view` — a scheduler
subject is, in every verified case, a `create_view_method = template` view, since a real scheduler
subject always needs a `UNION` of resource types and/or calculated HTML/colour columns that Meta
Auto/Meta Custom can't express. For control-procedure mechanics generally (code groups, static vs. SQL
assignment, `branch_rdbms_type`, dialect translation) see
`thinkwise_software_factory_create_control_procedures`; every control procedure referenced below
(the view's SELECT, update handlers, process-flow-called tasks) is created and generated exactly that
way. This skill only covers what's specific to the Scheduler.

## What a Scheduler component is

A Scheduler visualizes appointments or tasks on a timeline: **time cells** represent slices of time (an
hour, a day, a week…) and **activities** are appointments plotted against a **resource** — an employee,
a machine, a truck, a room. It is a **Universal UI-only** built-in component; the legacy Windows GUI
equivalent (**Resource Scheduler**) needs a hand-written object model extender and should be treated as
legacy for new work (see "Migrating from the legacy Resource Scheduler extender" below). A table or
variant gets a Scheduler by placing the **Scheduler** screen component in one of its screen types; the
component silently hides itself if no `scheduler` row exists for that table, or if placed in a Windows
GUI screen without the extender — an empty component is usually a sign the definition is missing, not
that something crashed.

## Recommended workflow — new Scheduler from scratch

1. **Interview the user on the Scheduler's core design before touching the model**: resource shape
   (single grouping column vs. a multi-level hierarchy, and if hierarchy, whether it spans one
   self-referencing table or several physically different tables — the latter needs the
   prefixed-synthetic-key technique below); which `scheduler_view`s are needed and their
   timescales/pagination/business-hours behaviour; whether date-dragging and/or resource-dragging
   should be enabled; whether external drag-and-drop and/or an Add-activity task are in scope, and
   if so, which columns on the subject become that task's parameters; and how the activity's
   title/tooltip should render — see "Ask before building: what should an activity look like?"
   under "HTML and multiline activity formatting" below for the exact question (and HTML-style
   follow-up) to ask.
2. **Present that plan and get the user's explicit confirmation before creating any
   `scheduler`/`scheduler_view` row.** This is the Scheduler-specific instance of the
   confirm-before-mutate / ask-don't-default rules in `thinkwise_software_factory_mcp_base`'s
   "Shared conventions" section — resource shape, view/timescale count, drag-drop scope, and
   Add-activity parameters are all real design decisions, not mechanical CRUD, so don't let staging
   begin section by section without a confirmed plan covering all of them first.
3. Only then proceed entity by entity: confirm/build the subject's data model (primary key,
   resource+activity union shape) → `scheduler` → `scheduler_view` row(s) →
   `scheduler_view_resource_col` → conditional layout → screen → tasks and process flows — each of
   the sections below, in order.

## Data model: the subject the Scheduler points at

The table or view the Scheduler is configured against (the **subject**) is the activity list — every
row is one activity, plus (optionally) resource-only rows with no activity. Get this right before
touching any Scheduler-specific configuration; the great majority of scheduler bugs (drag-drop 400
errors, appointments that silently fail to move, resources that don't group correctly) trace back to
the subject's data model, not the Scheduler settings layered on top of it.

**What the subject needs:**
- A column (or lookup) identifying the resource an activity belongs to.
- A start date/datetime column and an end date/datetime column — both nullable at the row level (see
  "Resource rows with no activity" below), even though the Scheduler configuration requires you to
  nominate both.
- Optionally a title column and a tooltip column.
- A primary key that is **stable, non-nullable, and does not shift under drag-and-drop.**

### Primary key: single non-nullable column, not a composite of resource + date

Give the subject its own surrogate identity column as the primary key — even when the activity
conceptually "belongs to" a resource, don't fold `resource_id` or a date column into the key. Dragging
an activity *updates* exactly those columns (resource-dragging rewrites the resource FK, date-dragging
rewrites the dates); a key that changes identity on every drag is a modeling smell and breaks anything
that references the row. **Verified live**: the `PROJECT_MANAGER` reference model's scheduler subject
(`project_planning_scheduler`) has `no_of_pk_col = 1`, `max_pk_col_no = 1` — a single-column primary
key (`resource_scheduler_id`, `varchar`, mandatory) — never a composite of resource and dates.

**Known issue — nullable primary keys.** A subject built from a `UNION` of several sources (leave
requests, sick leave, regular activities combined into one feed) tends to end up with a key column
that's `NULL` for some branches. Community reports have traced drag-and-drop 400 errors directly to
this, confirmed by Thinkwise as *"a nullable primary key is not supported."* Synthesize a non-null row
identifier per branch instead of trusting whichever source column happens to line up across all of
them.

### Technique: prefixed synthetic keys for a hierarchy spanning multiple tables

Verified, real technique from `PROJECT_MANAGER`'s subject — a **department → team → employee**
resource hierarchy built from three physically different tables, unioned into one view. Shape of the
pattern: one `SELECT … UNION ALL …` branch per level/table (department, team, employee, plus the
activity-bearing table), where every branch prefixes its native ID before it reaches
`resource_scheduler_id` (`d_`, `t_`, `e_`, `pt_`) — this is what lets one non-nullable `VARCHAR` primary
key column serve several different source tables without collision (`d_7` and `e_7` are obviously not
the same row). Each branch's `parent_resource` reuses the exact same prefixed format as the level above
it, which is all a hierarchy grouping needs (see "Resource grouping" below); the activity branch's
`resource_id` reuses the assigned employee's own prefixed key rather than minting a new one. Reach for
this whenever a resource hierarchy spans genuinely different tables rather than one self-referencing
one. For the full worked SQL (all four branches, verbatim), read
`references/hierarchy_key_example.md`.

**Bonus — this can eliminate the need for an instead-of trigger on resource-dragging.** Because
`resource_id` is the *same prefixed string* on the resource's own row and on every activity assigned to
it, dragging an activity onto a different resource just copies that string across verbatim — no FK
lookup translation required. Building the grouping key as a plain, format-matched string rather than a
raw numeric FK is a legitimate way to avoid the instead-of trigger described below entirely.

### Resource rows with no activity

To show a resource even when it currently has no appointments, include a row for it with an empty
start and end date. This is why start/end date are "required" only in the sense that the Scheduler
configuration needs you to nominate which columns play that role — at the row level they're nullable,
and a null start/end is exactly how "resource with nothing scheduled" is represented. Verified: every
resource-only branch in the `PROJECT_MANAGER` example above selects `null as title`, `null as
start_date`, `null as end_date`.

### If the subject is a view: reference direction and the instead-of trigger

Scheduler subjects are very often views. Two rules matter, both defer-linked:

- **Reference direction flips for views** — for an FK-shaped column on the view pointing at a real
  table's primary key, the real table is `source_tab_id` and the view is `target_tab_id` (the reverse
  of a normal table-to-table FK), with `check_ref = false` since a view carries no physical FK
  constraint. Full explanation and examples in `thinkwise_datamodeling_guidelines`.
- **Resource dragging needs an instead-of trigger on most view subjects.** Dragging an activity to
  another resource updates the "group by" column to the target resource's value; since lookup-value
  translation isn't applied automatically on that write, a view subject typically needs an
  *instead-of update* trigger translating the incoming value back into the correct underlying foreign
  key before it lands — unless the subject uses the prefixed-synthetic-key trick above, which sidesteps
  the need for one.

### Migrating off the legacy Resource/Task/Worktime model

The old Windows GUI Resource Scheduler extender pattern split **Resource**, **Task/Activity** and
**Worktime** into three separate subjects. The Universal Scheduler wants **one** subject where each
resource can produce multiple rows (one per activity) — exactly the `UNION ALL` shape above. There's no
first-class "worktime" concept anymore; model working/non-working time as ordinary activities styled
differently (see "'Work time' — a resource-availability/capacity pattern" under time-cell colouring
below), not as a parallel table. Full comparison
table and migration checklist in `references/legacy_migration.md`.

## `scheduler` — field reference (one row per table/variant)

Keyed by `(model_id, branch_id, tab_id)`.

| Column | Purpose |
|---|---|
| `type_of_resource_grp_by` (enum) | `single_col_grp_by` = 0 · `hierarchy_grp_by` = 1 |
| `resource_grp_by_col_id` | The resource grouping column |
| `parent_resource_grp_by_col_id` | Parent grouping column, hierarchy mode only |
| `hierarchy_default_expanded` (flag) / `hierarchy_default_expanded_level` | Whether hierarchy nodes start expanded, and how many levels deep |
| `allow_date_dragging` (flag) | Enables moving/resizing an activity within the same resource |
| `allow_resource_grp_dragging` (flag) | Enables reassigning an activity to a different resource by drag |
| `activity_title_col_id` / `activity_tooltip_col_id` | Title/tooltip columns |
| `activity_start_date_col_id` / `activity_end_date_col_id` | Start/end date columns |
| `activity_start_task_parmtr_id` | The **Add activity task**'s parameter that receives the clicked time cell's start date/time |

Verified real row (`PROJECT_MANAGER`, `project_planning_scheduler`): `type_of_resource_grp_by =
hierarchy_grp_by`, `resource_grp_by_col_id = resource_id`, `parent_resource_grp_by_col_id =
parent_resource`, `hierarchy_default_expanded = true` at 2 levels, `allow_date_dragging = true`,
`allow_resource_grp_dragging = true`, `activity_title_col_id = title`, `activity_tooltip_col_id =
description`, `activity_start_date_col_id = start_date`, `activity_end_date_col_id = end_date`,
`activity_start_task_parmtr_id = start_date`.

Bound tasks on `scheduler_view` (the sibling entity below carries the interesting ones — `scheduler`
itself is edited in place, no bound tasks beyond the standard history/unlink).

**Creating this row, verified**: a bare/unscoped "add" of a new `scheduler` record can be rejected
outright by the write API in use, with no indication in the error that routing is the problem (it can
read like a permission failure). The fix is to address the new record as a **detail of its owning
table's record** — i.e. navigate to the specific `tab` row first, then add the `scheduler` record
through that table's own detail relationship to `scheduler`, rather than creating it unscoped. Adding
it this way also auto-populates `tab_id` (and `model_id`/`branch_id`) from the parent context, so
there's no need to set `tab_id` by hand at all. If a metadata/introspection call on the API in use can
enumerate an entity's parent-relationship options, check there for the correct routing before assuming
a rejected unscoped add means a permissions problem.

## `scheduler_view` — field reference (one row per view)

Keyed by `(model_id, branch_id, tab_id, scheduler_view_id)`. A single Scheduler can offer several
views (Day/Week/Month-style), each with its own timescale, pagination behaviour, and cell styling.

| Column | Purpose |
|---|---|
| `order_no` | Sequence in the view switcher; the first one shown is used when no explicit default view is set |
| `show_scheduler_view` (flag) | Visibility toggle — hide a view without deleting it |
| `enable_sliding_window` (flag) | Off: a page covers the full span of the highest timescale (e.g. Jan 1–Dec 31 for a year view). On: the window centers on *today* instead (quarter/month starts one week in the past; year starts one month in the past) |
| `use_time_scale_year` / `_quarter` / `_month` / `_week` / `_day` / `_hour` / `_minute` (flags) | Which timescales are active on this view |
| `time_scale_*_interval` (int, one per timescale) | Interval per active timescale, e.g. every 2 hours |
| `show_label_lowest_time_scale` (flag) | Off: the lowest interval still slices the cells for fine-grained drag-drop, but its header labels are suppressed |
| `time_cell_min_width` (int, px) | Minimum cell width |
| `min_displayed_time` / `max_displayed_time` (time) | Business-hours clamp — hide hours outside this range |
| `hide_monday` … `hide_sunday` (flags) | Per-weekday visibility, day-timescale only |
| `day_label_format` (enum) | `day_no` = 0 · `day_no_and_name` = 1 |

**Modeling rule of thumb**: the **highest** enabled timescale becomes the page you paginate through;
the **lowest** becomes the individual cells; anything in between renders as an extra header row.
Configure at least two timescales per view.

**Critical**: working hours (`min_displayed_time`/`max_displayed_time`, `hide_monday`…`hide_sunday`)
are set **per Scheduler view**, globally — there is no per-resource working-hours setting. Model
per-resource variation as differently-styled activities or time-cell conditions instead (see below).

**Known limitation**: resources are always sorted **alphabetically** on the grouping column; there's no
sort-by-date/priority setting. A numeric prefix baked into the grouping/display value is the common
workaround.

Verified real example — three views on `PROJECT_MANAGER`'s one Scheduler:

| View | Timescales | Sliding | Hidden days | Hours shown | Day label |
|---|---|---|---|---|---|
| `month` | month(1) → week(1) → day(1) | on | none | all | number + name |
| `work_week` | week(1) → day(1) | on | Sat, Sun | all | number + name |
| `work_day` | day(1) → hour(1) → minute(15) | on | none | 07:00–18:00 | number only |

Bound tasks: `task_copy_scheduler_view`, `task_delete_scheduler_view`, `task_rename_scheduler_view`,
`task_show_history`, `task_unlink_generated_object`.

## `scheduler_view_resource_col` — resource panel columns

Keyed by `(model_id, branch_id, tab_id, scheduler_view_id, col_id)`. Shows extra, read-only
information alongside each resource in the grouping panel (an employee's role, a truck's capacity).

| Column | Purpose |
|---|---|
| `include_resource_col` (flag) | Whether the column is actually shown — an un-included row is configured but hidden |
| `order_no` | Display sequence |
| `col_width` (int, px) | Initial width; users can resize it afterwards (cached in the browser) |

Making one visible is conceptually three steps: locate the row, check `include_resource_col`, set
`order_no`/`col_width`. Point resource columns at a translated **look-up** value rather than a raw
foreign key or code — the panel is read-only real estate. Verified: all three of `PROJECT_MANAGER`'s
views expose exactly one resource column, `resource_name`, 250px wide.

**Locating the row, verified**: don't assume "locate" means "add" — through the API in use, a direct
add of a new `scheduler_view_resource_col` record (even when correctly routed as a detail of its
`scheduler_view`) was rejected, because the actual writable surface for this data exposes a
differently-named "overview" variant of the entity where **a row already implicitly exists for every
candidate column** on the view's subject, defaulting to not-included. The correct approach is to
address that existing row directly by its full key (`model_id`/`branch_id`/`tab_id`/
`scheduler_view_id`/`col_id`) as an **edit**, then set `include_resource_col = true` and the
`order_no`/`col_width` you want — not to add a new row. If an API's schema exposes both a plain-named
entity and an "overview"/similarly-suffixed variant for the same data, and a direct add on the plain
one fails, check whether the variant is the one actually meant to be written to.

Bound tasks: `task_show_history`, `task_unlink_generated_object`.

### Consider conditional layout for a new resource column — but only where it's warranted

After making a `scheduler_view_resource_col` visible, take one pass asking whether the column's value
is worth highlighting: a capacity/availability figure that can run low or over, a role/type that should
stand out, a status that means the resource can't currently take work. If so, the mechanism is an
ordinary table-level `conditional_layout` targeting that same `col_id` on the subject table, with
`apply_to_scheduler_resource = true` (see "Conditional layout — resources" below) — there is no
resource-column-specific conditional layout entity; it's the same table-level family as the rest of the
subject's columns.

**Only add one where there's a real candidate — don't add one just because a resource column exists.**
A plain, always-populated label like `resource_name` rarely needs styling; a capacity/availability
figure or an exception state often does. When there's no good candidate, say so and add nothing.

**Never add one without checking with the user first** — present the candidate column, the condition,
and what it would communicate, and get explicit confirmation before creating anything. If the Scheduler
is part of a larger plan, fold the candidate into that plan and get the **plan** confirmed before
finalizing it, not as a silent addendum once the Scheduler is being built.

For the actual mechanics — field reference, the condition enum, light/dark colours — see
`thinkwise_software_factory_conditional_layouts`. This note only decides *whether* one is warranted for
a resource column; that skill covers *how* to build it (and applies equally to activity-level
conditional layout on this same subject).

**A resource-column conditional layout only paints the views that actually display that literal
column.** Different `scheduler_view` rows on the same Scheduler can show a different column for what is
conceptually the same resource label — e.g. one view's `scheduler_view_resource_col` shows a raw code
column while another shows that code's looked-up/display variant in the same slot. Since a conditional
layout's `col_id` targets exactly one column, a layout built against the raw column has no visible
effect in a view that instead displays the lookup variant. Check every visible `scheduler_view`'s
resource-column set (not just one) before assuming one layout (or one set of colour layouts) covers the
whole Scheduler — a second full set targeting the other variant column is needed for full coverage.

## `scheduler_view_conditional_layout` / `_condition` / `_tag` — time-cell colouring

Keyed by `(model_id, branch_id, tab_id, scheduler_view_id, cell_color_id[, cell_color_no |
tag_id])`. Distinct from the ordinary `conditional_layout` entity used for activities/resources below —
this family colours the **time cells** of the grid itself.

`scheduler_view_conditional_layout` (the "cell colour"):

| Column | Purpose |
|---|---|
| `cell_color_description` | Name |
| `background_color_light` / `background_color_dark` (`Edm.Int32`) | Per-theme background colour |

`scheduler_view_conditional_layout_condition`:

| Column | Purpose |
|---|---|
| `type_of_time_scale` (enum) | `col` = 0 · `time_scale` = 1 · `date` = 2 |
| `time_scale` (enum, only when `type_of_time_scale = time_scale`) | `year`=0 · `quarter`=1 · `month`=2 · `week`=3 · `day`=4 · `hour`=5 · `minute`=6 |
| `col_id` (only when `type_of_time_scale = col`) | Column evaluated against the **resource** record |
| `condition` (enum, 18 operators) | `equal_to`=0 · `not_equal_to`=1 · `greater_than`=2 · `smaller_than`=3 · `greater_than_or_equal_to`=4 · `smaller_than_or_equal_to`=5 · `between`=6 · `starts_with`=7 · `contains`=8 · `does_not_contain`=9 · `is_empty`=10 · `is_not_empty`=11 · `does_not_start_with`=12 · `not_between`=13 · `ends_with`=14 · `does_not_end_with`=15 · `in`=16 · `not_in`=17 |
| `type_of_value` / `until_type_of_value` (enum) | `constant`=0 · `column`=1 |
| `value` / `until_value` / `value_col_id` / `until_value_col_id` | Constant or column-sourced comparison value(s) |
| `date_value` / `until_date_value` (datetimeoffset) | Exact UTC range, only for `type_of_time_scale = date` |

**When staging a write**, expect `type_of_time_scale`/`condition`/`type_of_value` to require the raw
numeric value rather than the string key shown above (e.g. `2` for `date`, not the string `"date"`) —
consistent with the general enum-key-rejection quirk noted in `thinkwise_datamodeling_guidelines`'s
API-write-quirks reference.

`scheduler_view_conditional_layout_tag`: a plain `(tag_id, value)` pair per cell colour, same tagging
mechanism used elsewhere in the model.

### Worked pattern: colouring cells for a varying, per-period resource state

The field reference above gives the raw enum shape but not the technique for the single most common
real use of this family: showing a state that comes and goes over time for a given resource — a
machine's planned downtime, an employee's vacation/sick leave, a truck's maintenance window — as a
coloured block on the time cells, distinct from any activity bar.

**Verified live**, on a real scheduler subject (`GREEN_FLOW`'s `Production_Planning_Tab_Task_POC`,
modeling resource "work time" windows): the subject's `UNION`-based query gets **one extra row per
state-period**, alongside its resource and activity rows — each such row shares the resource's own
grouping key (so it lands under the right resource) but leaves every activity-rendering column
(title, tooltip, and the start/end date columns `scheduler.activity_start_date_col_id`/
`activity_end_date_col_id` point at) `null`, so it never draws as an activity bar. It carries only its
own pair of period-start/period-end date columns and a state/type column.

For **each distinct colour/state**, model one `scheduler_view_conditional_layout` row with **two
AND'ed conditions**:
1. A `date`-type condition (`type_of_time_scale = date`) testing whether the cell's date falls
   `between` that row's own two period columns, with `type_of_value = column` on both bounds
   (`value_col_id`/`until_value_col_id` pointing at the period-start/period-end columns) — not a fixed
   `date_value`. This is what makes the colouring track each row's own dates instead of a constant range.
2. A `col`-type condition (`type_of_time_scale = col`) testing that same row's state/type column
   `equal_to` a constant identifying this specific colour.

Both conditions target columns on the **same underlying subject row** (the unioned period row), not
necessarily the resource's own header row — "column evaluated against the resource record" in the
field reference above means whichever row of the subject is being evaluated for that resource, which
for this pattern is the unioned period row.

Repeat the pair for every distinct state (one `scheduler_view_conditional_layout` per colour), and for
every `scheduler_view` that should show the colouring — a layout only applies to the view it's keyed
under.

### "Work time" — a resource-availability/capacity pattern, not a built-in mechanism

The legacy Windows GUI Resource Scheduler extender had a literal, first-class **Worktime** subject —
a separate table saying, per resource, which hours/days it's available (see "Migrating off the legacy
Resource/Task/Worktime model" above). **The Universal Scheduler has no equivalent built-in entity** —
the business need still comes up constantly, but has to be reconstructed with the plain data-modeling
and conditional-layout tools available, using exactly the union-per-period pattern above.

**Verified live shape**: a dedicated child table, keyed by its own identity, with a resource FK, a
period-start date column, a period-end date column, and a colour/state column — wired into the
scheduler subject exactly per the worked pattern above: unioned in as extra non-activity rows sharing
the resource's own grouping key, then coloured with one `scheduler_view_conditional_layout` per
distinct colour value that row can carry.

**Common uses of the same mechanism, different business meaning:**
- **Standard business hours / shift patterns per resource** — tint a resource's own working hours
  distinctly from its off-hours, when the Scheduler-wide `min_displayed_time`/`max_displayed_time`/
  `hide_*day` settings (identical for every resource) aren't granular enough for per-resource
  variation.
- **Planned downtime / maintenance windows** — for equipment, room, or vehicle resources: a block
  over the period a machine is offline for servicing.
- **Employee absence, vacation, sick leave** — the same technique applied to people instead of
  equipment.
- **Part-time / reduced-capacity periods** — a resource only available some days a week, or at
  reduced hours during a specific date range (e.g. a seasonal contract).
- **Public holidays** — a period that colours the same day across every resource at once, rather than
  one resource's own schedule.

**Design choice: a raw colour column vs. a named-state enum domain.** Storing a literal colour value
directly on each period row keeps the second condition a simple equality check against that colour
code, but a model with distinct named states (`vacation`/`sick_leave`/`maintenance`/`holiday`) is
usually clearer with a proper enum domain instead (see `thinkwise_datamodeling_guidelines`'s "Domain
elements" section) — one value per state, with the meaning explicit in the data rather than encoded as
a colour that only means something by convention.

**Critical gotcha — match the condition to the enabled timescales.** A condition on a timescale the
view doesn't include is always true. A view enabling only Year/Month/Day with an `hour > 8` condition
colours *every* cell, because there is no hour timescale to evaluate against — the condition must
also constrain at least the lowest timescale actually present.

**Known gap** — colouring specifically Saturday/Sunday via a time-scale condition on day-of-week isn't
directly supported; Thinkwise's stated workaround is custom CSS, alongside the coarser
`hide_saturday`/`hide_sunday` flags on `scheduler_view` as a partial substitute.

**Verified**: this family is genuinely optional — a scheduler can rely entirely on activity-level HTML
styling (below) instead, and `PROJECT_MANAGER`'s scheduler did exactly that across all three of its
views for a long time. It later gained `scheduler_view_conditional_layout` rows specifically to colour
employee absence/vacation periods (see the worked pattern above) — don't assume every real scheduler
uses this family, but don't assume none ever will either.

Bound tasks (on `scheduler_view_conditional_layout`): `task_copy_scheduler_view_conditional_layout`,
`task_delete_scheduler_view_conditional_layout`, `task_rename_scheduler_view_conditional_layout`,
`task_show_history`, `task_unlink_generated_object`.

## Resource grouping

| Type | Setup |
|---|---|
| **Single column** | One resource column; rows sharing a value group together (e.g. group trucks by `truck_type`) |
| **Hierarchy** | A group-by column and a parent group-by column; every parent must also exist as its own resource row; configure default-expanded state and how many levels deep |

For a hierarchy spanning more than one physical table, use the prefixed-synthetic-key technique above
rather than assuming hierarchy grouping requires a single self-referencing table.

## Screen setup

Verified real configuration (`PROJECT_MANAGER`'s `project_planning_scheduler` `tab` row) — the pattern
to replicate for any new Scheduler:

| `tab` field | Value | Why |
|---|---|---|
| `main_screen_type_id` / `detail_screen_type_id` | both a screen type containing **only** the Scheduler component | Keeps the Scheduler as the sole focus of the screen |
| `max_no_of_records` / `page_size` | `0` / `0` | Disables platform grid pagination so the Scheduler's own windowing controls what loads — normal pagination fights the component otherwise |
| `allow_add` / `allow_copy` / `allow_delete` | `false` | Mutation happens through drag-drop/tasks, not the standard record CRUD buttons |
| `allow_update` | `true` | **Required** — drag/drop and resize write through this |
| `use_update_handlers` | `true` | Needed for a view subject to accept the drag/drop writes |

## Update handlers, resizing, and drag-drop

Dragging, resizing and reassigning activities are all the same mechanism: the Scheduler's update
handler writes new start date / end date / resource values straight into the subject's underlying
table (or, for a view, through whatever instead-of trigger sits behind it). Two prerequisites gate all
of it:

1. **Update permission** on the subject (off by default for views — a very common reason drag-drop
   silently does nothing).
2. The two toggles on `scheduler`: `allow_date_dragging` (move within the same resource) and
   `allow_resource_grp_dragging` (reassign to a different resource) — both default to on.

**Resizing** is date-dragging applied to one end only — if only one of the start/end date parameters is
wired on the related task/handler, resizing only works from that one edge.

**Resource dragging** rewrites the grouping column to the target resource's value; on a view subject
this needs an instead-of trigger unless the prefixed-synthetic-key trick (above) is in play. Before
reporting drag-drop as "broken," check, in order: (1) Update permission on the subject, (2) whether the
subject is a view needing an instead-of trigger, (3) whether the primary key contains a nullable
column.

**Only the columns `scheduler` actually configures** (`resource_grp_by_col_id`,
`activity_start_date_col_id`, `activity_end_date_col_id`) are guaranteed to reflect a drag/resize's
outcome inside the update handler. Other update-handler-enabled columns on the subject still arrive as
ordinary handler parameters, but nothing guarantees they're fresh on a drag — don't derive a
write-critical value (a foreign key, say) from one of those instead of from the configured column.
**Caution, not independently verified against a live drag** — a reasoned inference from the handler's
parameter contract, flagged here so it gets tested rather than assumed the first time it matters.

## External drag-and-drop

Users can drag rows from an unrelated grid/tree (a backlog of unassigned orders) onto a time cell to
create an activity from them, via a **drag-drop link** (`drag_drop` entity family — the full field
reference for `drag_drop`/`drag_drop_parmtr`/`drag_drop_matrix`, including the `drop_behavior`
enum and the variant-combination matrix, lives in `thinkwise_software_factory_subject_components`;
this section only covers what's specific to a Scheduler as the drop target):

1. On the source subject, define a drag-drop link: source tab, target tab (the Scheduler's subject),
   and a **Drag-drop task** run on drop.
2. Map **Drag-drop parameters** — source column → task parameter.
3. Because the target is a Scheduler, a **Drop date time parameter** field appears; picking it
   auto-populates the Scheduler's own `activity_start_task_parmtr_id`.
4. Set the Scheduler's **Add activity task** to the same drag-drop task so click-to-create and
   drag-to-create share logic.
5. Enable the interaction (`Enable drag-drop`) — it's off by default.

Dragging multiple selected rows fires the task once **per row**, not a single batched call; parameters
shared between source and target are validated for equality on drop — a mismatch silently blocks the
drag rather than erroring.

## Click-to-create and double-click → popup

### Click-to-create (Add activity task)

Build a table task with parameters for everything the new activity needs, set it as the Scheduler's
**Add activity task**, and pick a **Start date time parameter** — the clicked cell's date/time is
passed in automatically (`scheduler.activity_start_task_parmtr_id`). **Verified**: `PROJECT_MANAGER`
uses `project_planning_scheduler_add_activity`, a plain `STORED_PROCEDURE` table task, wired to
`activity_start_task_parmtr_id = start_date`.

**Other parameters can auto-populate from the clicked row too, not just the start date.** The start
date/time binding above is the one field the Scheduler special-cases explicitly
(`activity_start_task_parmtr_id`); a task parameter with default-input enabled and named identically to
a column on the subject should also inherit that column's value from the clicked row through the
platform's ordinary column-to-parameter default-binding mechanism (the same one table Defaults use) —
useful for passing along which resource was clicked, not just when. **Not independently verified
against a live click** — reasoned from the platform's general default-input mechanism; test it against
an actual click before relying on it silently working.

**A parameter that should render as a look-up in the task's input form isn't automatically one just
because its domain matches a real table's primary key.** Model a task-level reference for it (the task
equivalent of a table's `ref`) pointing at the source table, the same way an FK-shaped view column
needs its own `ref`/`ref_col` before it gets look-up behavior (see `thinkwise_datamodeling_guidelines`).

**Known limitation**: no built-in way to disable "Add activity" for specific resources/resource
groups. Workaround: a default value or a process-flow check that blocks execution and shows a message.

**Hide the task's own display**: an Add-activity task is meant to be triggered only by the click,
never as a manual toolbar button sitting next to it — set the table task's display type to hidden
(leave it enabled/shown as a table task otherwise; hiding only its button rendering doesn't disable
the click-to-create binding, which fires independently of button visibility).

**The new Add-activity task (and its parameters) needs translating too.** Like any newly created
task, it's created with a bracket-placeholder translation (`[employee_schedule_add_activity]`) —
that's what shows on the table-task button/toolbar until it's translated, not a broken label.
`scheduler_view` rows (e.g. `month`, `work_week`) get the same placeholder treatment. Follow
`thinkwise_software_factory_translation_objects` after wiring up the Scheduler to catch these
alongside the underlying subject's own table/column labels.

### Double-click — two valid routes

1. **Simple route — direct task.** Table task with `grid_double_click = true` on the Scheduler's
   subject. **Verified**: `PROJECT_MANAGER`'s own activity double-click,
   `project_planning_scheduler_open_activity_detail`, is exactly this — a plain `STORED_PROCEDURE`
   table task, no process flow at all. Use this when double-click just needs to run logic or navigate.
2. **Popup route — DUMMY task + process flow.** Use this when you specifically want a modal detail view
   without leaving the Scheduler screen. Full recipe below.

### The DUMMY-task + process-flow + popup recipe

**1 — Create a DUMMY task.** `task_type_id = 'DUMMY'` — no SQL of its own; its only job is to be
something a grid can double-click and a process flow can declare as its starting point. **Verified**:
`PROJECT_MANAGER`'s Scheduler toolbar buttons `project_planning_scheduler_go_to_date` and
`project_planning_scheduler_go_to_next_week` are both `task_type_id = DUMMY`.

**2 — Wire it up.** Attach the dummy task as a table task on the Scheduler's subject; check
**Double click on record** (`tab_task.grid_double_click = true`) for a double-click trigger, or
`show_tab_task = true` alone for a toolbar button.

**3 — Build the process flow.** Name it by the model's own convention — `pf_<task_id>` verified in
this model — check **User action** (`process_flow.use_starting_points = true`), add the dummy task as
its starting point, and lay out `start` → *actions* → `stop`.

**4 — The actions.** For a jump-to-date flow (**verified**, `pf_project_planning_scheduler_go_to_date`):
`start` (98) → `execute_tab_task` (6, runs the DUMMY task to collect a date) → **`activate_scheduler`
(790)**, a dedicated Scheduler process action that jumps the Scheduler to that date → `stop` (99). For a
double-click-to-detail flow: `start` → **`change_filter`** (330, filters the target table to the
double-clicked row's key) → **`open_document`** (2, opens a **dedicated table variant** rather than the
default) → `stop`.

**5 — Make "Open document" render as a popup, not a navigation.** Point step 4's `open_document` at a
table variant whose `main_screen_type_id`, `detail_screen_type_id`, `zoom_screen_type_id` **and
`popup_screen_type_id`** are all set to the same lightweight screen type. **Verified**: the flow
`open_employee_calendar_item` (`start` → `change_filter_employee_calendar_item` (330) →
`open_document_employee_calendar_item` (2, `tab_variant_id = form_only`) → `stop`) targets a
`form_only` variant with exactly this shape. Skip the dedicated `popup_screen_type_id` and "Open
document" just does a normal full-screen navigation instead of a modal.

Relevant `process_action_type` codes, verified:

| Action | Code |
|---|---|
| `start` | 98 |
| `stop` | 99 |
| `execute_tab_task` (Start table task) | 6 |
| `change_filter` (Change filters) | 330 |
| `open_document` (Open document) | 2 |
| `activate_scheduler` (Activate Scheduler) | 790 |

**Note**: in the live reference model, `open_employee_calendar_item` is actually invoked via
`process_flow.use_api_trigger = true` from a custom FullCalendar component rather than a grid
double-click — the change-filter/open-document/popup-variant mechanism is identical regardless of
trigger; combine it with steps 1–2 above to get the double-click version.

**Tip — more than one action per appointment.** Double-click only gives one destination. If users need
to choose between several actions (edit, cancel, duplicate…), route the double-click into a process
flow with a chooser step instead of cramming branching logic behind a single click.

**Gotcha**: if a task parameter comes through empty on double-click, check the underlying view's join
before touching the task/parameter mapping — a real case traced an empty parameter to the view not
joining on both the resource and the activity id.

## Conditional layout — activities

An ordinary `conditional_layout` row on the Scheduler's subject, scoped to a column (e.g. status) — no
separate "activity colour" concept. Set `show_conditional_layout = true`, pick the column (blank =
whole row), configure background/font colour per theme plus bold/italic/underline/strikethrough and
size, then add `conditional_layout_condition` rows. For the full field reference, condition enum, and
known gaps/pitfalls of this entity family, see `thinkwise_software_factory_conditional_layouts` — this
section only covers what's Scheduler-specific (the HTML interaction below, and the resource variant
next).

**Don't combine with HTML formatting** — if the title/tooltip column has HTML/Multiline control turned
on, conditional layout is ignored for that activity, and specifically font-size/strikethrough/underline
are ignored even in mixed setups. Do all styling inline in the HTML/CSS once you're in HTML territory.
This is exactly why the reference model still keeps one plain conditional layout around (`text_white`,
on `title`, white background both themes, `apply_to_scheduler_resource = false`) — as a fallback for
the one activity display type (below) that opts *out* of HTML.

## Conditional layout — resources

Same mechanism, plus the `apply_to_scheduler_resource` checkbox — colours the resource's row/label in
the grouping panel rather than an activity bar. Typical use: grey out an unavailable resource, tint
parent vs. child rows in a hierarchy. **Evaluation rule**: only the first matching record per resource
decides which layout applies — design conditions assuming "first match wins."

**Don't** expect this to apply alongside HTML-formatted activities/tooltips — mutually exclusive in
practice, same as above.

**Known gap**: no native resource capacity/availability concept to colour against (unlike the old
extender's Worktime table). Workaround: synthetic "off-duty" activities styled distinctly, or a
time-cell condition off explicit hour columns. The reference model's `number_of_hours_available`/
`number_of_hours_planned` columns are exactly the kind of data you'd condition on — in that model
they're randomly generated demo placeholders (`cast((abs(checksum(newid())) % 13) + 5 as int)`), not a
working capacity engine; in production these would come from a real rostering/availability source.

## HTML and multiline activity formatting

### Ask before building: what should an activity look like?

Before setting a control type on the title/tooltip column, ask the user directly (`AskUserQuestion`)
how they want an activity to render — don't default to HTML just because it's the richest option:

- **Single line** — plain text, one line, no wrapping. Domain control type stays plain/default.
- **Multiline** — plain text with line breaks, domain control type `MULTILINE`. Good for a title plus
  a short subtitle with no colour/styling need. **Verified**: `PROJECT_MANAGER`'s tooltip column
  (`description`) uses domain `scheduler_tooltip`, control `MULTILINE`, plain text.
- **HTML** — rich formatting (colour swatches, badges, progress bars, cards), domain control type
  `HTML`, backed by a calculated column building an HTML string with `concat`. **Verified**:
  `PROJECT_MANAGER`'s `title` column uses domain `title_html`, control `HTML`. Remember this also
  silences conditional layout on that column (font-size/strikethrough/underline specifically) — see
  "Don't combine with HTML formatting" above.

**If HTML is chosen, ask a follow-up** on which display style to use — present the verified styles
below from the reference model's 10-element `activity_display_type` dispatcher domain (see "The
dispatcher pattern" in `references/html_activity_formatting.md` for the SQL behind each) rather than
leaving the user to invent one from scratch:

| Style | Looks like | Best for |
|---|---|---|
| `outlook_second_line` | Bold title, italic subtitle line underneath | Title + secondary detail, no colour coding needed |
| `status_round` | Filled circle swatch + title | Compact status colour cue |
| `status_rounded_rectangle` | Rounded-square swatch + title | Same cue, slightly more visible swatch — **recommended default** |
| `status_slim_bar` | Thin vertical colour bar + title | Mimicking Outlook/Google Calendar's colour-strip convention |
| `status_outlined` | Outlined circle (border colour only) + title | Status cue without implying a solid/committed state (e.g. "tentative") |
| `pill` | Title + small rounded badge | A flag/count/label that needs extra emphasis (e.g. "!") |
| `progress_bar` | Title + percentage slider below | Progress/completion tracking |
| `activity_card` | Multi-line card: eyebrow label, title, avatar + name | Rich detail when cell height allows it |
| `activity_card_compact` | Same card, no avatar row | Rich detail in tighter cells |
| `none` | Plain title, no HTML wrapper | Opting out of HTML for this activity — pair with an ordinary conditional layout instead |

**Recommendation**: default to `status_rounded_rectangle` when the user has no strong preference — it
reads clearly at any cell width or zoom level, conveys a status colour without needing the extra row
height a card needs, and reuses the same `activity_status_color` computation as most of the other
styles (only `progress_bar` and `activity_card`/`_compact` need columns beyond that). Reach for
`status_slim_bar` instead when the user specifically wants the Outlook/Google-Calendar colour-strip
look, `pill` when there's a count/flag to surface, and `activity_card`/`_compact` when the Scheduler's
row height is generous enough for multi-line content.

Once the format (and, for HTML, the display style) is confirmed, proceed with the mechanics below.

For the dispatcher pattern that drives HTML activity layouts from a domain column (status colour,
progress bar, activity cards, pill badges, tooltip emoji/unicode, and height management to stop tall
HTML content from blowing out row height), read `references/html_activity_formatting.md` before
implementing.

## Migrating from the legacy Resource Scheduler extender

The old Windows GUI Resource Scheduler extender (three separate subjects: Resource, Task/Activity,
Worktime; hand-written object model extender code; a zoom slider instead of named views) maps onto the
Scheduler component's concepts one-for-one, but not automatically. For the full concept-by-concept
comparison table and the step-by-step migration checklist, read `references/legacy_migration.md` before
starting a migration.

## Bound-task quick reference

| Entity | Bound tasks |
|---|---|
| `scheduler_view` | `task_copy_scheduler_view`, `task_delete_scheduler_view`, `task_rename_scheduler_view`, `task_show_history`, `task_unlink_generated_object` |
| `scheduler_view_resource_col` | `task_show_history`, `task_unlink_generated_object` (no copy/rename/delete — see the "Locating the row" note above: the row already exists implicitly for every candidate column and is edited in place, not deleted and re-added) |
| `scheduler_view_conditional_layout` | `task_copy_scheduler_view_conditional_layout`, `task_delete_scheduler_view_conditional_layout`, `task_rename_scheduler_view_conditional_layout`, `task_show_history`, `task_unlink_generated_object` |

`task_copy_scheduler_view` takes `from_tab_id`/`from_scheduler_view_id`/`to_tab_id`/
`to_scheduler_view_id` — useful for cloning a working timescale/pagination setup (e.g. `work_week`) onto
a new table rather than re-typing every flag.

## Known pitfalls (verified)

- **Nullable primary keys on `UNION`-based subjects break drag-and-drop** with a 400 error — synthesize
  a non-null, prefixed row identifier per union branch instead.
- **Resource dragging on a view subject needs an instead-of trigger** unless the resource key is a
  format-matched string shared verbatim between resource rows and activity rows.
- **Update permission is off by default for views** — the most common reason drag-drop silently does
  nothing.
- **A time-scale condition on a timescale the view doesn't enable is always true** — colours every
  cell, not none.
- **HTML/Multiline formatting silences conditional layout** on that activity (specifically font-size,
  strikethrough, underline) — style inline in the HTML/CSS instead.
- **No per-resource working hours** — `min_displayed_time`/`max_displayed_time`/hide-weekday are set
  per `scheduler_view`, globally.
- **Resources always sort alphabetically** on the grouping column — no date/priority sort setting.
- **No native resource capacity/availability concept** — model it via synthetic activities or explicit
  hour columns feeding a time-cell condition.
- **`max_no_of_records`/`page_size` must both be `0`** on the subject — otherwise platform pagination
  fights the Scheduler's own windowing.
- **No built-in way to disable Add-activity for specific resources** — gate it in the task/process flow,
  not by trying to suppress the click.
- **No single-resource calendar/day view, and no Gantt dependency lines** — both are deliberate product
  gaps; a custom component (e.g. FullCalendar-based, verified live as `employee_calendar_item` in the
  reference model) is the sanctioned route for either.
- **A rejected unscoped/top-level "add" of a new `scheduler` record is not necessarily a permissions
  problem** — before concluding access is missing, retry it addressed as a detail of its owning
  table's record instead of as a standalone create. A rejected top-level add and a routing problem can
  look identical from the error alone.
- **`scheduler_view_resource_col` is not directly addable, even correctly routed as a detail of its
  `scheduler_view`** — the writable surface is an "overview"-style variant of the entity with a row
  already implicit for every candidate column; locate and edit that existing row by its full key
  instead of adding a new one.

## Pre-flight checklist

- **Every Scheduler design choice needs the user's explicit confirmation before it's built, not just
  resource-column conditional layout.** Resource shape (single-column vs. hierarchy), which
  `scheduler_view`s/timescales to build, drag-drop scope (date/resource/external), and Add-activity
  task parameters are all real design decisions — see "Recommended workflow" above and
  `thinkwise_software_factory_mcp_base`'s "Shared conventions" section (confirm-before-mutate /
  ask-don't-default). The conditional-layout-specific case below is one recurring instance of this,
  not the only one.
- Confirm the connector's actual domain keys for scheduler / data model / control procedures / tasks /
  process flows before assuming `manage_scheduler`-style names hold — check `manage_datamodel` too for
  `scheduler`/`scheduler_view_conditional_layout*` if the first search comes back empty.
- Create the `scheduler` record as a detail of its owning table's record, not as an unscoped/top-level
  add — the latter can be rejected outright even with correct permissions. For
  `scheduler_view_resource_col`, check whether the API exposes an "overview"-style variant with rows
  already implicit per candidate column — if so, edit the existing row by its full key rather than
  adding one.
- The subject's primary key is a single, non-nullable column — never resource + date, never a raw
  union-inherited column that goes `NULL` on some branches.
- If the subject is a view: confirm reference direction is reversed (real table = source, view =
  target, `check_ref = false`) per `thinkwise_datamodeling_guidelines`, and confirm whether resource
  dragging needs an instead-of trigger (it doesn't, if resource keys are format-matched strings).
  Actually writing/generating that view's SELECT follows `thinkwise_software_factory_create_view`
  end to end, including checking `branch_rdbms_type` before any dialect-specific SQL.
- `main_screen_type_id`/`detail_screen_type_id` point at a Scheduler-only screen type, and
  `max_no_of_records`/`page_size` are both `0`.
- At least two timescales are enabled per `scheduler_view`, and any `scheduler_view_conditional_layout_condition`
  constrains at least the lowest timescale actually present on that view.
- Update permission is enabled on the subject before expecting drag-drop to do anything.
- HTML/Multiline-controlled title/tooltip columns don't also rely on conditional layout for
  font-size/strikethrough/underline — those are silently ignored once HTML is in play.
- **The activity's display format (single line / multiline / HTML) is asked, not assumed** — and if
  HTML, the specific display style is confirmed against the recommended-default table before building
  the dispatcher SQL. See "Ask before building: what should an activity look like?" above.
- **Only add a conditional layout on a resource column where there's a real candidate, and confirm it
  with the user before creating anything** — don't default to one just because a resource column is
  visible, and don't finalize a plan that includes one without that confirmation. See
  `thinkwise_software_factory_conditional_layouts` for the mechanics.
- A DUMMY task used as a process-flow trigger carries no SQL/business logic of its own — that belongs
  in the flow's `change_filter`/`open_document`/`activate_scheduler` actions and their control
  procedures.
- A popup-style "Open document" target has its own `popup_screen_type_id` set on the variant being
  opened — without it, "Open document" navigates instead of popping up.
- The Add-activity task, its parameters, and each `scheduler_view` are translatable objects like any
  other — they start with bracket-placeholder text (see
  `thinkwise_software_factory_translation_objects`), not a real label.
