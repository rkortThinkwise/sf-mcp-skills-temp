---
name: thinkwise-software-factory-variants
description: Reference guide for creating, configuring, and maintaining table, task, and report variants in a Thinkwise Software Factory model — the field-level-vs-snapshot inheritance mechanism and the Changes-compared-to-default/where-used mechanics. Use whenever creating, inspecting, or troubleshooting a variant via an MCP connector with Software Factory access, and before deciding whether a request needs a new variant at all versus a prefilter, a new screen type, or a separate table/task/report.
---

# Creating and Maintaining Variants in the Thinkwise Software Factory

A **variant** is an alternative presentation or configuration of an existing table, task, or report
— never a copy of its data or logic. A table variant still queries the same table/view; a task
variant still executes the same task procedure and parameter set; a report variant still runs the
same report. What changes is screen type, columns, permissions, defaults, wording, icon, and similar
presentation/context settings — never the underlying query, business logic, or authorization.

```text
Default subject, task, or report
              |
              +-- Variant A: selected overrides
              +-- Variant B: selected overrides
              +-- Variant C: selected overrides
```

Apply this whenever an MCP connector with Software Factory access is used to create, inspect, or
modify a variant of any kind — follow the connector's standard discovery→act flow; never guess
entity/task/property names. Everything below was confirmed live against a connected model (domain
key `sf/manage_datamodel`) — try it directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session. This skill
is connector-agnostic: it names entities, tasks, and fields, not any one connector's literal tool
names.

**Companion skills, not duplicated here**: full task-variant mechanics (`task_variant`,
`task_variant_parmtr`, `task_variant_look_up_overview`, creation via `task_create_task_variant`) live
in `thinkwise_software_factory_tasks` — read it before building a task variant; this skill covers
only how a table variant *selects* a task variant (`tab_variant_task_overview`). Screen-type
assignment fields (`main_screen_type_id` etc.) always point at a screen type that already exists —
creating a new one isn't supported through the MCP tools; see
`thinkwise_software_factory_build_planner`'s interaction-surface step and the screen-type API-quirk
note in `thinkwise_datamodeling_guidelines`. Conditional-layout-per-variant mechanics live in
`thinkwise_software_factory_conditional_layouts`. Prefilter states and groups live in
`thinkwise_software_factory_prefilters`. Scheduler and map per-variant settings live in
`thinkwise_software_factory_scheduler_component` and `thinkwise_software_factory_maps_component`.
Unit-testing guidance (what to test, what not to) lives in
`thinkwise_software_factory_unit_tests`.

## Golden rule — this is a judgment call skill, not just a mechanics reference

The mechanics of building a variant are simple API calls. The hard part — and the actual point of
this skill — is **deciding whether a variant is the right tool at all**, which one of the three
kinds to use, and how much of it to override. Whenever the guidance below doesn't make the answer
obvious, **stop and ask the user** rather than silently picking the option that sounds safest. This
applies especially to:

- Whether the request needs a variant at all, or is better served by a plain (unlocked) prefilter, a
  screen-type change on the default, or a genuinely separate table/task/report (see "Variant versus
  another mechanism" below).
- Which of the three variant kinds — table, task, report — actually matches what's being asked.
- Whether a described difference is "presentation/context" (variant-safe) or "different data, logic,
  or side effects" (needs a real separate object, not a variant).
- The variant's **name**, when the model's existing naming pattern is ambiguous or inconsistent.
- Any **delete**, or a change that resets/overrides something a menu item, detail, lookup, process
  flow, or drag-and-drop link currently depends on — confirm intent first (see "Track every
  reference" below).
- Whether a **set-level aspect** (grid, form, card list, tree, detail, filter, search, or sort) needs
  a Setup snapshot at all, versus staying inherited from the default. Taking that snapshot is a real,
  semi-durable design commitment (see "Set-level settings" below) — confirm it with the user before
  calling any `task_setup_tab_variant_*_overview` task; this is not a call the assistant should make
  unilaterally.

Only skip asking when the answer is genuinely unambiguous — e.g. the user names the exact variant to
reuse or extend, or the difference is trivially and entirely a presentation change.

## The three variant kinds — object graph

| Kind | Master entity(ies) | Presentation entity | Key | Created via |
|---|---|---|---|---|
| Table (subject) variant | `tab` (default) | `tab_variant_overview` | `tab_id, tab_variant_id` | `task_create_tab_variant` (bound to `tab` or `tab_variant_overview`) |
| Task variant | `task` (default) | `task_variant`/`task_variant_overview` | `task_id, task_variant_id` | `task_create_task_variant` — see `thinkwise_software_factory_tasks` |
| Report variant | `tab_report`/report master (default) | `report_variant` + `report_variant_overview` | `report_id, report_variant_id` | Plain `stage_resource` add against `report_variant` — **no dedicated create task exists** (confirmed live: `search_domain_capabilities` surfaces no `task_create_report_variant`); it's a bare-key entity add, the same pattern the tasks skill documents for creating a `task` row itself |

A table variant is by far the most structurally rich of the three — it carries not just its own
scalar settings but a full family of child `tab_variant_*_overview` entities, one per configurable
aspect of the table (columns, grid, form, tasks, reports, prefilters, look-ups, details,
conditional layouts, cubes, scheduler, maps, card list, tree, sort, filter, search).

### Before creating: confirm the plan

Before the first `task_create_tab_variant`/`task_create_task_variant`/`report_variant`-add call,
lay out the plan in one short message and get the user's confirmation on it — the create call itself
is cheap to undo, but the design decisions behind it (see the Golden rule above) are not ones to
guess at. This is the `thinkwise_software_factory_mcp_base` "Confirm-before-mutate" convention applied
at this skill's own grain:

- Which of the **three kinds** (table, task, report) this is.
- The intended **name**.
- Which **fields** will be overridden (field-level settings) and which **sets** (grid, form, card
  list, tree, detail, filter, search, sort) will get a Setup snapshot versus staying inherited.
- Any **variant→variant chaining** (a look-up or detail pointed at a specific variant of another
  table) this variant will introduce or depend on.

Only skip this confirmation when the user has already stated all of the above unambiguously.

### Creating, copying, renaming, deleting a table variant

Go through the dedicated bound tasks — keys aren't renamable in place, the same pattern as
tables/columns/tasks elsewhere in the model:

- `task_create_tab_variant` — mandatory `tab_id`, `tab_variant_id`, `generate_transl_object`;
  optional `tab_variant_description`. Leave `generate_transl_object` on unless the variant is
  deliberately meant to share the default table's translation.
- `task_copy_tab_variant` — `from_tab_id`, `from_tab_variant_id`, `to_tab_variant_id`. Copies an
  existing variant's full configuration as a starting point for a new one (e.g. cloning a
  structurally similar variant onto the same or a different table).
- `task_rename_tab_variant` — `tab_id`, `from_tab_variant_id`, `to_tab_variant_id`.
- `task_delete_tab_variant` — `tab_id`, `tab_variant_id`. **Check `tab_variant_used` first** (see
  "Track every reference" below) — deleting a variant that a menu item, detail, lookup, or process
  flow still points at breaks that reference.

Once created, a table variant's own row (`tab_variant_overview`) already carries a full set of
scalar fields — screen types, CRUD/grid/form behaviour flags, icon, page size, badge, offline
settings, and more (see the field-level list below) — every one of them independently overridable,
no activation step needed.

## Inheritance — the mechanic that actually matters, verified live

Inheritance is not a documentation-only concept here — it's implemented by two genuinely different
mechanisms depending on the kind of setting, and the difference is directly visible in the model's
own entities.

### Field-level settings — always independently overridable, no snapshot

Plain scalar/flag settings on `tab_variant_overview` itself (`main_screen_type_id`,
`detail_screen_type_id`, `zoom_screen_type_id`, `popup_screen_type_id`, `icon_id`, `page_size`,
`allow_add`/`allow_update`/`allow_delete`/`allow_copy`, `max_no_of_records`,
`no_of_fields_locked`, `no_of_cols_in_form`, `badge_interval`, and similar) can each be changed on
the variant independently — changing one has no effect on any other, and every one of them
continues to silently track the default table's current value **until the variant's own value is
set to something different**. There is no explicit "activate this override" step for these.

**Only override `icon_id` (here, and on the equivalent `task_variant_overview`/
`report_variant_overview` fields below) when the variant's outcome or context genuinely differs from
the default** — per `thinkwise_software_factory_icons`. If a variant just prefills different
parameters for the same underlying object, leave `icon_id` tracking the default and let translation
carry the distinction instead; don't set an override as a matter of course just because the field is
there.

The same field-level pattern holds for the per-row child entities that exist automatically for
every column/task/prefilter/report/etc. once a variant is created — confirmed live:
`tab_variant_col_overview` gets one row per `(col_id, rdbms_type)` combination on the default table
the instant the variant is created, before any user edits anything. Verified on a live connected
model: for a table with 25 real columns, a brand-new-looking variant's `tab_variant_col_overview`
already had one row per column, and every row's `type_of_col` exactly matched its own
`default_type_of_col` — the platform-maintained mirror of the default table's current `col.type_of_col`
for that RDBMS. **`type_of_col == default_type_of_col` means the variant is still inheriting; a
divergence between them is the actual override.** Changing the default table's column type later
updates `default_type_of_col` (and, for a still-inheriting row, `type_of_col` alongside it); once a
variant sets its own `type_of_col`, the two values diverge and the variant stops tracking that
column's default type.

### Set-level settings — require an explicit "Setup" step to snapshot

A second family of child entities represents an **ordered collection** rather than one independent
value per row: grid columns, form columns, card list, tree, details, filter, search
(`combined_filter`), and sort. These are confirmed live to have a dedicated **Setup** bound task on
`tab_variant_overview` that plain field-level settings do not. This skill covers the lifecycle
mechanic (Setup/Reset/inheritance) generically across all of them; for what each field in the
grid/form/card-list/tree sets actually configures and renders, see
`thinkwise_software_factory_subject_components`:

| Setup task | What it snapshots |
|---|---|
| `task_setup_tab_variant_grid_overview` | Grid column set/order |
| `task_setup_tab_variant_form_overview` | Form column set/order |
| `task_setup_tab_variant_card_list_overview` | Card list layout |
| `task_setup_tab_variant_tree_overview` | Tree layout |
| `task_setup_tab_variant_detail_overview` | Detail tab set/order |
| `task_setup_tab_variant_filter_overview` | Filter field set |
| `task_setup_tab_variant_search_overview` | Search field set |
| `task_setup_tab_variant_combined_filter_overview` | Combined filter/search set |
| `task_setup_tab_variant_sort_overview` | Default sort |

**`task_setup_tab_variant_tree_overview` does not auto-populate the hierarchy link, verified live.**
Running it creates the expected `tab_variant_tree_overview` row per column, but for a self-referencing
hierarchical tree it leaves `parent_col_id` unset on all of them — the tree renders with no rows/no
hierarchy even though `tree_type`/`tree_display_col_id` are already correctly set on
`tab_variant_overview`. Fix it by explicitly setting `parent_col_id` on the **identity/PK column's**
`tab_variant_tree_overview` row to the self-referencing parent-FK column (e.g. on the `department_id`
row, set `parent_col_id = 'parent_department_id'`) — after that the tree renders correctly. Always
verify `tab_variant_tree_overview` after running the Setup task for a hierarchical tree rather than
assuming the wizard wired the hierarchy end-to-end. **Note the direction here is inverted from the base
table's own `col.parent_col_id`**, which lives on the FK column and points *at* the PK column (the
opposite row/value pairing) — see `thinkwise_software_factory_subject_components` for the base-table
mechanism and this asymmetry side by side.

Before Setup is called for a given set, that set has no independent identity of its own — the
variant is purely following the default's current grid/form/etc. Calling Setup takes what is
effectively a snapshot: from that point, the set stops automatically absorbing changes made to the
default. If a column is later added to the default table, an already-snapshotted grid or form on the
variant will generally **not** pick it up automatically — it stays hidden/absent until someone
deliberately adds it to the variant's own set. This is exactly the maintenance trade-off the research
literature describes: a snapshot protects a deliberately designed variant from accidental drift, but
it also means every future addition to the default has to be reviewed against every snapshotted
variant.

**Every one of the 18+ overview kinds also has a `task_reset_tab_variant_<x>_overview`** (confirmed
live for `col`, `grid`, `form`, `card_list`, `tree`, `detail`, `filter`, `search`,
`combined_filter`, `sort`, `task`, `report`, `prefilter`, `look_up`, `map`, `scheduler`, `cube`,
`conditional_layout`, and the field-level settings themselves via
`task_reset_tab_variant_overview`) — reset discards the variant's own values for that aspect and
resumes pure inheritance from the current default. Use Reset, not manual field-by-field
undo, whenever the intent is "stop this variant from diverging on X" rather than "diverge
differently."

Prefilters are explicitly **not** part of the snapshot mechanism — a `tab_variant_prefilter_overview`
row exists per prefilter automatically (same pattern as columns), and a newly added default
prefilter becomes visible to a variant without any Setup/reset step, unless the variant explicitly
overrides that prefilter's state.

### `tab_variant_change` — the live "Changes compared to default" comparator

`tab_variant_change` (bound task `task_reset_tab_variant_change` per row) is not a stored table —
it's a computed comparison, confirmed live: querying it for a variant returns exactly the settings
that currently differ from the default, each with `variant_col_value` and `default_col_value`
side by side (e.g. `main_screen_type_id`: variant `form_detail_no_action` vs. default `customer`;
`allow_add`: variant `False` vs. default `True`). Settings that still match the default —
verified: an unchanged `detail_screen_type_id` simply doesn't appear in the result at all. This is
the mechanism behind the platform's "Compare a variant to its default" screen, and it's the
authoritative way to audit a table variant's field-level overrides, since `tab_variant_overview`
itself carries no per-field "is this overridden" indicator the way `tab_variant_col_overview` does
with `default_type_of_col`. Query it after any bulk copy, before a release, and whenever a variant
looks like it's drifted further from its stated purpose than intended.

## Attaching tasks, reports, look-ups, and details to a table variant

Once a table variant exists, it already has one row per task/report/prefilter/look-up/detail the
default table has — the same auto-population pattern as `tab_variant_col_overview`. Configure the
variant's presentation by patching these rows, not by re-adding them:

| Overview entity | Key | What it overrides |
|---|---|---|
| `tab_variant_task_overview` | `+task_id` | `show_tab_task`, **which `task_variant_id` shows** for this variant, icon, group, screen area, primary action, display type, order |
| `tab_variant_report_overview` | `+report_id` | `show_tab_report`, which `report_variant_id` shows, icon, group, screen area, display type, order |
| `tab_variant_look_up_overview` | `+ref_id` | Look-up control type, popup, **which `look_up_tab_variant_id`** the reference resolves to, visibility for filter |
| `tab_variant_detail_overview` | `+ref_id` | `show_detail`, order, screen area, **which `detail_tab_variant_id`** renders the detail, alt. translation for the reference-add label |
| `tab_variant_prefilter_overview` | `+tab_prefilter_id` | `main_prefilter_state`/`detail_prefilter_state`/`look_up_prefilter_state` (off/off_hidden/on/on_locked/on_hidden), group, order |
| `tab_variant_conditional_layout_overview` | see `thinkwise_software_factory_conditional_layouts` | Per-variant conditional formatting |
| `tab_variant_scheduler_overview` / `tab_variant_map_overview` / `tab_variant_cube_overview` | see their component skills | Per-variant scheduler/map/cube view selection |

**This is the one-`tab_task`-row-per-table constraint's resolution, confirmed by design**: `tab_task`
itself only allows one row per `(tab_id, task_id)` — it cannot carry two task variants of the same
task on the same table. Showing two task variants side by side means giving them **separate table
variants**, each independently picking its own `task_variant_id` via its own
`tab_variant_task_overview` row. The same logic applies to `tab_variant_report_overview` for report
variants.

**Look-up and detail chaining**: `tab_variant_look_up_overview.look_up_tab_variant_id` and
`tab_variant_detail_overview.detail_tab_variant_id` let one table variant point a reference or
detail at a *specific variant* of the target table, rather than its default. This is how a
"compact lookup" or "portal detail" variant on table B gets wired into a variant of table A — but
per the research, avoid chaining variant→variant→variant more than one or two levels deep before it
becomes difficult to trace which effective configuration actually renders at runtime.

## Report variants — creation and settings

`report_variant` itself is a bare add — no dedicated bound task exists to create one (confirmed via
`search_domain_capabilities`: no `task_create_report_variant` surfaces, only `Reset`/`unlink` tasks
touch existing rows). Add it directly against `report_variant` (`report_id`, `report_variant_id`),
then configure the presentation entity `report_variant_overview` (same key), which carries:

| Field | Purpose |
|---|---|
| `report_action` | What happens on run — verified enum: `none`, `print_preview`, `print`, `export_to_PDF`, `export_to_RTF`, `xml`, `csv`, `xls`, `xls_data_only`, `xlsx_data_only`, `doc`, `word_rtf`, `xlsx`, `docx`, `tiff`, `html`, `png` |
| `printer_id` | Target printer, for a variant meant to print rather than preview/export |
| `type_of_communication` | Await-result mode — `synchronous`, `async`, `synchronous_with_progress` (default), `synchronous_with_progress_possible_async` |
| `display_parmtr_id` | Which report parameter's value is shown as the variant's display label |
| `confirm_button_has_alt_transl`/`alt_transl_confirm_button`, `cancel_button_has_alt_transl`/`alt_transl_cancel_button` | Per-variant Confirm/Cancel wording — leave off unless genuinely different from the default report |
| `icon_id`, `show_badge`/`badge_interval`, `no_of_cols_in_form`, `form_col_min_width_factor`, `conditional_layout_code` | Presentation, same pattern as table/task variants |
| `generate_transl_object` | Keep on unless deliberately sharing the base report's translation |

This is the mechanism behind the `print`/`preview`/`export_pdf`/`export_excel`-style variant families
— same report content and data selection, different `report_action`/`printer_id`/parameter defaults.
If the content, template, or underlying dataset must differ, that's a separate report, not a variant
(see below).

## Deciding when to use a variant — and when not to

A good variant describes one stable audience, workflow, or entry point in a single sentence:
*"Shop-floor view of released production operations, opened from the workstation menu, locked to the
current workstation, card layout, start/pause/complete/material-registration tasks only."* If it
can't be summarized that concisely, it may be trying to be a genuinely separate object.

| Requirement | Prefer |
|---|---|
| Same data and logic, different screen presentation | Table variant |
| Same task logic, different defaults or wording | Task variant |
| Same report, different parameters or output behavior | Report variant |
| User can optionally toggle a filter | A normal (unlocked) prefilter — not a variant |
| Dedicated menu worklist with a locked context | Table variant (with a locked/hidden prefilter) |
| Reusable component arrangement across unrelated subjects | An existing screen type (creating a new one isn't supported via MCP — see `thinkwise_software_factory_build_planner`) |
| Different SQL dataset or joins | A separate view/table |
| Different business logic or side effects | A separate task |
| Different report content or template | A separate report |
| Different authorization | Roles and access control — a variant can only make rights *more* restrictive, never grant something the default/role denies |
| Tenant isolation | Access-control and data-isolation logic, not a variant |
| Different navigation, branding, or deployment | Separate application configuration |
| Personal presentation preference | User personalization, where available |

**Avoid creating a variant when**: the underlying query must genuinely differ; the action has
different business semantics; the report needs a different template or dataset; it would be the
*only* control on who can do what (rights must be enforced independently); the only real difference
is an optional filter a normal prefilter would cover; it requires overriding almost every setting
(a sign the "variant" is actually a separate object in disguise); it exists only to avoid fixing a
poor default; or a near-identical variant already exists with purely cosmetic differences.

## Naming and documentation

Name variants by **purpose and audience**, not by number or implementation detail:

```text
production_order_shop_floor
production_order_planning_scheduler
production_order_history
certificate_customer_portal
customer_active_lookup
material_registration_receipt
picking_list_export_pdf
```

Avoid `variant_1`, `new_screen`, `test`, `copy`, `version_2`, `alternative` — names that describe
neither purpose nor implementation. Give every variant a description covering its intended audience,
entry points, key prefilters, permission restrictions, why the default doesn't already serve this
need, and whether it carries a grid/form/detail snapshot.

## Maintenance

- **Override the minimum.** Every override is an exception to inheritance; a sparse variant keeps
  benefiting from default improvements. Before taking any set-level snapshot (Setup task), **ask the
  user** whether the set genuinely needs to be stable independently of the default — if not, leave it
  inheriting.
- **Review `tab_variant_change` regularly** — after adding columns/tasks/reports to the default,
  after screen-type changes, and before every release. Reset any override that turns out to be
  accidental rather than intentional.
- **Track every reference before deleting or radically changing a variant.** `tab_variant_used`
  (key `pk_col`, carrying `tab_id`/`tab_variant_id` plus a `type_of_object` enum spanning the entire
  model's object catalog) is the live "Applied to" list — confirmed to cover menu items, details,
  look-ups, table tasks/reports, process actions (including `process_action_start_tab_variant` /
  `process_action_start_report_variant` process-flow starting points), and drag-and-drop links. A
  variant that looks unreferenced in one modeler screen may still be wired into a process flow —
  query this entity rather than assuming.
- **Avoid deep variant chains.** A table variant can point its look-ups and details at *other*
  table variants (see chaining above), which can themselves apply task/report variants. Keep each
  link in the chain purposeful and clearly named; a long chain becomes very hard to reason about at
  runtime.
- **Keep rights explicit.** The effective behaviour is role rights AND default rights AND variant
  restrictions AND prefilter/data-isolation logic, all at once. A variant that hides a menu item or
  task is not the same as securing it — an unbound or unassigned task is still reachable via the API
  regardless of what any variant shows.

## Testing

Per `thinkwise_software_factory_unit_tests`: unit-test the underlying task/default/layout/handler
logic once — variants reuse that same logic, so they don't each need their own copy of the same
test. The one exception is a **task variant that silently supplies a default** (e.g. a
`material_receipt` variant defaulting `transaction_type = RECEIPT`) — that hidden branch is real
business logic and deserves its own unit test on the underlying task. Beyond that, variant
correctness is a configuration/user-flow concern: verify each important variant opens from its
intended entry point, applies its prefilters, shows the right columns/details, hides the tasks/
reports it should, supplies correct defaults, and enforces the intended rights — prioritizing
customer portals, shop-floor/scanner workflows, financial approval screens, and any variant with a
large snapshot or a multi-level chain.

## Pre-flight checklist

- **Ask before deciding** whether a variant is warranted at all, which kind, and its name — per the
  Golden rule above — whenever the framework doesn't make it obvious.
- **Create in the right order**: `task_create_tab_variant` (or the equivalent for task/report
  variants) first; only then patch the auto-populated child `*_overview` rows.
- **Field-level settings need no activation** — just patch the value; it stops inheriting the
  moment it diverges from the default (verified via the `default_*` mirror columns where present,
  e.g. `tab_variant_col_overview.default_type_of_col`).
- **Set-level settings (grid/form/card list/tree/detail/filter/search/sort) need `task_setup_*`
  before they're a genuinely independent snapshot** — don't assume patching rows alone detaches them
  from the default; and don't take the snapshot unless the set truly needs to be stable
  independently.
- **Use `task_reset_tab_variant_<x>_overview` to intentionally return to inheritance** — it exists
  for every overview kind, not just the snapshot ones.
- **Query `tab_variant_change` to audit drift** before a release or after any bulk copy — it's a
  live comparison, not a stored table, and only lists genuine differences.
- **One `tab_task`/`tab_report` row per table** — two task/report variants on the same table need
  two table variants, each with its own `tab_variant_task_overview`/`tab_variant_report_overview`
  selection, not two base rows.
- **Report variants have no dedicated create task** — add `report_variant` directly, the same
  bare-entity-add pattern the tasks skill documents for the `task` master row.
- **A variant restricts, never grants.** Never treat hiding something in a variant as equivalent to
  securing it — enforce that independently via roles/rights.
- **Check `tab_variant_used` before deleting** — a variant that looks unused in one screen may still
  back a process-flow start, drag-and-drop link, or lookup elsewhere.
- **Prefer a plain unlocked prefilter over a variant** when the only requirement is an optional
  toggle — reserve variants for a genuinely dedicated audience/workflow/entry point.
