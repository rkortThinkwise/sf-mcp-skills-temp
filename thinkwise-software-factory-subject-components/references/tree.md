# Tree view (Treeview)

## What it is, and when to use it

A Treeview presents one subject as either a **Hierarchical** tree (real parent-child rows, variable
depth) or an **Attribute** tree (fixed-level grouping by sorted columns — Thinkwise's own enum name;
community material calls this a "Column tree"). Users expand/collapse with the arrow keys or `LEFT`/
`RIGHT`; an active row stays expanded; Expand all/Collapse all are built in. The Universal UI hides the
component entirely when the subject has at most one row.

| Question | Hierarchical | Attribute (column-grouped) |
|---|---|---|
| Are parent nodes real, selectable rows? | Yes | Usually no — they're synthetic group nodes |
| Can depth vary per branch? | Yes | No — levels are fixed by which columns are sorted |
| Is there a parent identifier? | Yes — a self-referencing FK | Not required |
| Levels come from | Row relationships | Sorted/grouped columns, via **Group until** |

Use **Hierarchical** for bills of material, org charts, folder structures, category trees, work
breakdowns — anywhere depth genuinely varies and a parent is a real, addressable record. Use
**Attribute** grouping for `Country → Province → City → Customer`-style fixed-depth drill-downs where
the "parent" is just an attribute value, not a row anyone edits directly.

**Don't use a Tree** for essentially flat data, for records with more than one legitimate parent
(duplicating a node under several branches breaks identity/editing), or when users mainly need to
compare many columns across many rows (that's a Grid).

## Field reference — verified

`tab` / `tab_variant_overview` (subject-level; identical field set on both):

| Field | Type | Purpose |
|---|---|---|
| `tree_display_col_id` | lookup → `col` | Text shown for the node. Blank falls back to the subject's look-up display column. |
| `tree_image_col_id` | lookup → `col` | Icon shown before the text. |
| `tree_type` | enum: `hierarchical`=0, `attribute`=1 | Which tree mode. |
| `tree_default_expanded` | flag | Whether nodes start expanded. |
| `tree_default_expanded_level` | byte | How many levels deep, when expanded by default. |
| `tree_show_label` | flag | Whether the textual label renders alongside the node (vs. icon-only). |

Column-level hierarchy link, on plain `col` (not a tree-specific entity — this is the same field used
for other parent/child column relationships generally):

| Field | Type | Purpose |
|---|---|---|
| `parent_col_id` | lookup → `col` (same table) | Set **on the self-referencing foreign-key column** to name which column it points back to. This is what makes a Hierarchical tree work. |

**Verified live example** (`INSIGHTS.employee`, a manager/direct-report hierarchy):

```text
col employee_id: primary_key = true
col manager_id:  foreign_key = true, parent_col_id = "employee_id"
ref ref_employee_employee: source_tab_id = employee, target_tab_id = employee, check_ref = true
```

`manager_id` is a genuine self-referencing FK (backed by a real `ref` row, `check_ref = true`) **and**
carries `parent_col_id = employee_id` — the marker that tells the Hierarchical tree "this column is the
parent link, and this is the column it resolves to." Both parts matter: the physical self-reference
gives you the FK/lookup behavior everywhere else in the app; `parent_col_id` is what the Tree
specifically consumes to build the hierarchy. This matches (and sharpens) community guidance that
current tree modeling "identifies the parent relationship through the configured parent columns" rather
than needing a hand-built recursive query.

**Attribute (column-grouped) trees use ordinary column settings, not tree-specific ones**: mark the
grouping columns `default_sort = true` with ascending `sort_no`, and set `grp_until = true` on the last
column that should form a grouping level (everything sorted up to and including it becomes a tree
level; anything sorted after it is leaf-level ordering only). No live example of `tree_type = attribute`
was found in the connected repository during verification — treat the mechanism above as confirmed from
the field/enum shapes and cross-checked against community documentation, but it's less battle-tested
here than the Hierarchical path.

## Variant overrides

`tab_variant_tree` / `tab_variant_tree_overview`, keyed by `(…, tab_id, tab_variant_id, col_id)`:

| Field | Purpose |
|---|---|
| `parent_col_id` | Per-variant hierarchy link. **The direction is inverted from the base table's `col.parent_col_id` — verified live, see below.** |
| `type_of_col` / `calculated_field_type` (overview only) | Read-only context copied from `col`. |

**Gotcha — `parent_col_id` points the opposite way on `tab_variant_tree` vs. plain `col`.** On the base
table, `parent_col_id` is set *on the FK column's* row, and its value names the PK column it resolves to
(the verified `employee`/`manager_id` → `employee_id` example above). On `tab_variant_tree`, it's the
other way round: set it *on the identity/PK column's* row, and its value names the self-referencing FK
column that supplies the parent. **Verified live** (`RK_SCHEDULER_TEST.department`, `org_chart`
variant): the table has three columns (`department_id` PK, `department_name`, `parent_department_id`
FK) with a `tab_variant_tree` row seeded for each, but only the **`department_id`** row carries a value
— `parent_col_id = 'parent_department_id'` — while `parent_department_id`'s own row is blank. Don't
carry the base-table direction over to the variant field by assumption; they are not symmetric.

**Don't assume a variant's tree "just works" after `task_setup_tab_variant_tree_overview`** either —
per `thinkwise_software_factory_variants`, that Setup task creates one `tab_variant_tree_overview` row
per column but does not auto-populate `parent_col_id` on any of them. Read it back after Setup and set
it explicitly, on the identity/PK column's row, per the direction above.

## Step by step

1. **Hierarchical**: confirm (or create) a self-referencing `ref` on the subject (per
   `thinkwise_datamodeling_guidelines`), then set `parent_col_id` on the FK column to the primary key
   column it resolves to.
   **Attribute**: set `default_sort = true` + ascending `sort_no` on every grouping column, and
   `grp_until = true` on the last one that should form a level.
2. On `tab` (or the variant's `tab_variant_overview`), set `tree_type`.
3. Pick `tree_display_col_id` — prefer a human-readable, sibling-unique value; build a cheap expression
   column dedicated to tree presentation if no single column reads well alone. If no column obviously
   fits, ask the user rather than defaulting to whichever column merely looks human-readable.
4. Optionally set `tree_image_col_id` — reinforce meaning with text/status too, never icon-only for
   critical distinctions.
5. Set `tree_default_expanded` / `tree_default_expanded_level` conservatively (1–2 levels for most
   trees) and `tree_show_label`.
6. Place a Tree view leaf panel on an existing screen type for the subject — creating a new screen
   type isn't supported via MCP (see `thinkwise_software_factory_build_planner`).
7. If this is a variant, run `task_setup_tab_variant_tree_overview`, then explicitly verify/set
   `tab_variant_tree.parent_col_id` **on the identity/PK column's row**, pointing to the FK column —
   the opposite direction from step 1's base-table convention (see the gotcha above).

## Table vs. view for the tree's subject

Use the table directly when the parent-child relation lives in one entity and users edit nodes
normally. Use a view when several entity types must appear in one tree, a bill of material must be
exploded recursively, or icons/labels/paths need normalizing across sources — building it as a view
follows `thinkwise_software_factory_create_view`, and a multi-type hierarchy view is easiest to reason
about with a normalized shape (`node_id`, `parent_node_id`, `node_type`, `display_text`, `icon`,
`sort_order`).

## Best practices and pitfalls

- **Stable keys only.** Never build `parent_col_id`'s relationship from a mutable/translated label.
- **No cycles, no self-parenting.** The Tree component doesn't police this — enforce it in the task or
  handler that changes the parent.
- **No orphaned children.** If row-level security/prefilters can hide a parent while a child stays
  visible, decide deliberately: show the parent as structural context, hide the child too, or promote
  it to a visible root in a presentation view.
- **Conservative default expansion.** Levels of 10/20/99 are a maintenance smell unless the dataset is
  guaranteed tiny — Expand all/Collapse all already cover the rest.
- **Display column must earn its place.** Never a bare technical ID; keep any dedicated expression
  column cheap since it evaluates per node.
- **Gotcha — no dedicated conditional-layout flag for Tree.** `thinkwise_software_factory_conditional_layouts`
  confirms `apply_to_grid`/`apply_to_form`/`apply_to_edit`/`apply_to_scheduler_resource` as the complete
  set of surface flags — there is no `apply_to_tree`. Whether Tree nodes inherit `apply_to_form`'s
  styling, `apply_to_grid`'s, both, or neither has not been independently confirmed against a live
  rendered Tree — test before relying on conditional formatting showing up in a Tree.
- **Attribute trees**: keep the sort a business-meaningful, stable order — don't sort by translated
  display text when the group order matters, since translations can reorder groups per language.
