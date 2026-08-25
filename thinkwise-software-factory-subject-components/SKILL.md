---
name: thinkwise-software-factory-subject-components
description: Reference guide for the screen components that present a Thinkwise Software Factory subject (table or view) — Grid, Form, Card List, Treeview, and Drag-and-drop — covering the tab/col-level settings (tree_*/card_list_*/form_*/grid_* fields), their tab_variant_*_overview per-variant counterparts, and the drag_drop/drag_drop_parmtr/drag_drop_matrix family. Use whenever an MCP connector with Software Factory access (sf_mcp, indicium) is setting up or troubleshooting a subject's Grid/Form/Card List/Tree/Drag-drop behavior — before calling get_entity_definition/execute_odata_query/stage_resource against tab, col, tab_variant_grid/form/tree/card_list(_overview), or drag_drop*.
---

# Subject Components in the Thinkwise Software Factory: Grid, Form, Card List, Treeview, Drag-and-drop

Every one of these five components is driven by settings that live directly on the subject's `tab`
row (subject-level) and its `col` rows (column-level), plus a per-variant shadow of each in the
`tab_variant_overview` / `tab_variant_grid(_overview)` / `tab_variant_form(_overview)` /
`tab_variant_tree(_overview)` / `tab_variant_card_list(_overview)` family. None of the five is a
separate "component entity" the way `scheduler` or `map` are — the component only ever *renders* what
these plain subject/column fields already say, once it's placed as a leaf panel on a screen type that
already exists — creating a new screen type isn't supported through the MCP tools (see
`thinkwise_datamodeling_guidelines`'s quirks section and `thinkwise_software_factory_build_planner`'s
interaction-surface step).

Every entity, field, and enum below was confirmed live via `get_entity_definition` against a connected
Software Factory repository (`sf/manage_datamodel` domain), and every numbered example below was
pulled live from real models in that repository (`INSIGHTS`, `PROJECT_MANAGER`, `SQLSERVER_SF` — the
Software Factory's own self-hosted model) — not guessed from documentation. Community/documentation
research supplied by the user was used for the *when-to-use* and best-practice framing, then checked
against the live metadata; anywhere the two disagreed, this skill follows the verified metadata and
says so explicitly.

Apply this whenever an MCP connector with Software Factory access is used to create, inspect, or
troubleshoot a subject's Grid/Form/Card List/Tree/Drag-drop behavior. `tab`, `col`, every
`tab_variant_*` sibling below, and `drag_drop*` typically live in a `manage_datamodel`-style domain —
try it directly first, and only escalate to `search_capabilities`/`get_available_domains` on an
`entity_set_not_found`/`domain_not_found`-style rejection rather than re-discovering a domain that
already resolved earlier this session.

## How this skill relates to the others

This skill only covers what's specific to these five components' own settings. For everything around
them:

- **Placing the component itself** (which leaf panel goes where in the screen's split/tab tree,
  action bars, breakpoints) — this means picking an existing screen type, not creating one; see
  `thinkwise_software_factory_build_planner`'s interaction-surface step for how to choose among what
  already exists (and when to ask the user), and the screen-type API-quirk note in
  `thinkwise_datamodeling_guidelines`.
- **The variant lifecycle** these settings shadow into (`task_setup_tab_variant_*_overview` /
  `task_reset_tab_variant_*_overview`, inheritance-until-Setup, where-used) —
  `thinkwise_software_factory_variants`. This skill only calls out what's specific to *these five*
  overview families (e.g. the Tree Setup gotcha below); the general mechanic lives there.
- **Conditional formatting** on Grid/Form cells and rows — `thinkwise_software_factory_conditional_layouts`.
  That skill's verified `apply_to_grid`/`apply_to_form`/`apply_to_edit`/`apply_to_scheduler_resource`
  flags notably have **no `apply_to_card_list` or `apply_to_tree`** — see the gotcha in each of those
  two sections below.
- **Scheduler-specific drag-and-drop** (date/resource dragging, external grid-to-scheduler drop,
  `activity_start_task_parmtr_id`) — `thinkwise_software_factory_scheduler_component`. The generic
  drag-drop family below is what backs *external* drag-onto-scheduler; date/resource dragging on an
  *existing* scheduler activity is a completely separate mechanism (plain Update permission +
  `scheduler.allow_date_dragging`/`allow_resource_grp_dragging`), not part of `drag_drop*` at all.
- **Map marker dragging** — `thinkwise_software_factory_maps_component`. Also not part of
  `drag_drop*`: it's the `allow_drag_drop` flag on the map's data-mapping entity.
- **Data-modeling conventions** (naming, reference direction, self-references) —
  `thinkwise_datamodeling_guidelines`. The hierarchical Treeview's parent-column mechanism (below)
  leans directly on a self-referencing `ref`, which that skill covers generically.

## Shared foundation: the four-layer stack

```text
screen_type (leaf panel: Grid | Form | Card list | Tree view)   ← existing screen type only; see thinkwise_software_factory_build_planner
  └─ tab / tab_variant_overview          (subject-level: which column is title/image/tree-root, sizing, behaviour flags)
      └─ col / tab_variant_<x>(_overview) (column-level: per-component order, type, width, grouping)
          └─ tab_variant_<x>_overview adds: move up/down, renumber, copy-from-other-component, and
             read-only type_of_col/calculated_field_type context — never re-derive these by hand
```

A component can be fully, correctly configured on `tab`/`col` and still show nothing, because the
screen type it's meant to appear on doesn't contain that leaf panel — or the reverse: a panel is
placed but the subject has no meaningful configuration behind it (an un-ordered Card List, a Tree with
no display column). Always check both ends before troubleshooting deeper.

**Settings exist on every subject even when the component isn't used.** `tree_display_col_id`,
`card_list_title`, and friends are ordinary columns on every `tab` row — their presence doesn't mean a
Tree or Card List is actually shown anywhere; check the effective screen type first.

**Variant shadow, briefly** (full mechanic in `thinkwise_software_factory_variants`): `tab_variant_overview`
mirrors nearly every subject-level field below field-for-field; `tab_variant_grid` / `tab_variant_form`
/ `tab_variant_tree` / `tab_variant_card_list` mirror the column-level fields, keyed by
`(…, tab_id, tab_variant_id, col_id)`. Each has an `_overview` sibling (`tab_variant_grid_overview`,
etc.) that adds `type_of_col`/`calculated_field_type` (read-only context copied from `col`) plus bound
tasks — `task_move_<x>_col_up`/`_down`, `task_renumber_tab_variant_<x>`, and, for Grid/Form
specifically, **`task_copy_grid_to_form`/`task_copy_form_to_grid`** (copies order number and/or
visibility from one component to the other in one call — a genuinely fast way to seed a Form from an
already-tuned Grid or vice versa). A variant's grid/form/tree/card-list only has independent values
after its `task_setup_tab_variant_<x>_overview` has run; before that it's pure inheritance from the
table's own default. **Known gap, verified in `thinkwise_software_factory_variants`:
`task_setup_tab_variant_tree_overview` does not auto-populate the hierarchy link** — after Setup,
explicitly check/set `tab_variant_tree.parent_col_id` yourself (see the Tree section).

## Confirm the design before touching tab/col

This skill inherits `thinkwise_software_factory_mcp_base`'s "Shared conventions" — confirm-before-mutate
and ask-don't-default — for every `stage_resource`/`patch_resource` call below; see that section for the
general rules. What's specific to these five components: the choice that needs stating and confirming is
almost always a single up-front *mode* pick with real behavioural consequences, not a field-by-field
detail, so it's easy to fold silently into the first `tab`/`col` patch if you're not deliberate about it.

Before the first `tab`/`col` write for a new or changed component, state the chosen mode in plain
language and get the user's explicit confirmation — for example: Hierarchical vs. Attribute for a
Treeview, the Grid mode (read-only overview / deliberately editable / default-editable / add-in-Grid /
analytical), the Card List's title/image source, or Group vs. Section usage in a Form. The existing
cross-reference to `thinkwise_software_factory_build_planner`'s interaction-surface step (in "How this
skill relates to the others" above) only covers *where* the component is placed — which screen type it
lands on. It says nothing about *how* the component itself should behave, which is this skill's own
in-scope design surface and needs its own confirmation before any tab/col patching starts.

This includes the recurring case of adding one new column to a table whose Form already has
established groups/sections — deciding which group it joins (or that it needs a new one) is exactly
this kind of design choice, not a mechanical default to skip past.

---

# 1. Tree view (Treeview)

For Tree view setup (Hierarchical vs. Attribute mode, the tab/col-level `tree_*` fields, the
`tab_variant_tree.parent_col_id` direction gotcha, step-by-step instructions, table-vs-view guidance,
and pitfalls), read `references/tree.md`.

---

# 2. Card List

For Card List setup (tab/col-level `card_list_*` fields, step-by-step instructions, and pitfalls), read
`references/card_list.md`.

---

# 3. Form

For Form setup (tab/col-level `form_*` fields, Groups vs. Sections, variant overrides, step-by-step
instructions, and pitfalls), read `references/form.md`.

---

# 4. Grid

For Grid setup (tab/col-level `grid_*` fields, step-by-step instructions, and pitfalls), read
`references/grid.md`.

---

# 5. Drag-and-drop

For Drag-and-drop setup (the `drag_drop`/`drag_drop_parmtr`/`drag_drop_matrix` family, step-by-step
instructions, and pitfalls), read `references/drag_drop.md`.

---

## Pre-flight checklist

- Confirm the connector's actual domain key for the datamodel (`sf/manage_datamodel`-style) before
  assuming it holds everywhere; escalate to `search_capabilities` only on an outright rejection.
- **Tree**: confirm which mechanism applies — Hierarchical needs a self-referencing `ref` *and*
  `col.parent_col_id` on the FK column; Attribute needs `default_sort`/`sort_no`/`grp_until` on the
  grouping columns. After a variant's `task_setup_tab_variant_tree_overview`, explicitly verify
  `tab_variant_tree.parent_col_id` — Setup does not populate it for you, **and it belongs on the
  identity/PK column's row pointing at the FK column, the reverse of the base-table direction.**
- **Card List / Tree**: neither has a dedicated `conditional_layout` surface flag
  (`apply_to_card_list`/`apply_to_tree` don't exist) — verify empirically before assuming conditional
  formatting will show up on either.
- **Form**: set `form_type_of_col` on every field deliberately (never leave it to accident); treat
  Group (`form_field_in_next_grp`) and Section (`field_on_next_tab`) as genuinely different tools, not
  interchangeable ways to add visual space; don't hand-set the likely-computed `col.section` field
  directly without checking it's actually writable.
- **Grid**: `auto_resize_grid_col = no` once a table/variant reaches ~20 visible columns; distinguish
  header groups (`grid_field_in_next_grp`, cosmetic) from runtime row grouping
  (`default_sort`/`grp_until`/`grp_box_visibility`, structural); aggregate only same-unit measures.
  `task_copy_form_to_grid`/`task_copy_grid_to_form` are the fast path to keep a Grid and Form in sync.
- **Drag-and-drop**: a link is disabled in `drag_drop_matrix` by default for every variant combination
  — enable only the specific combinations that make business sense; use `drop_behavior = check_equal`
  for any compatibility check that should block an invalid drop; never let the task trust the drag
  alone for validation.
- Whichever component you're touching, verify the change against the effective **screen type** the
  subject/variant actually points at — a correctly configured subject/column setting is invisible if
  the leaf panel isn't placed, or is placed on a screen type the table doesn't use. Remember the
  screen type itself must already exist — creating a new one isn't supported via MCP (see
  `thinkwise_datamodeling_guidelines` and `thinkwise_software_factory_build_planner`).
