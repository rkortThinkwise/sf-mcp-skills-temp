---
name: thinkwise-software-factory-menu
description: Reference guide for creating and maintaining application menus, groups, and items — including per-role visibility — in a Thinkwise Software Factory model. Use via an MCP connector with Software Factory access whenever creating, inspecting, or securing a menu, and before deciding whether a new screen/report/task belongs on an existing menu/group or needs a new one.
---

# Creating and Maintaining a Menu in the Thinkwise Software Factory

Reference for the menu's full shape: `menu` (container: one `menu_type` — list bar / tile / tree —
serving one or more platforms via `menu_platform`) → **group** (`list_bar_grp` / `tile_grp` /
`module_grp` — a labelled section) → **item** (`list_bar_item` / `tile` — points at exactly one
table, table variant, report, report variant, task, or task variant) → per-role visibility
(`role_menu_overview` / `role_list_bar_grp_overview` / `role_list_bar_item_overview` /
`role_tile_grp_overview` / `role_tile_overview`). Every entity, field, enum value, and task signature
below was confirmed live against a real connected model (`sf/manage_menu` domain, model `INSIGHTS`) —
not guessed from documentation.

Apply this whenever an MCP connector with Software Factory access is used to create, extend, reorganize,
or secure a menu — follow the connector's standard discovery→act flow; never guess entity/task/property
names. `menu` and its groups/items/role-overview entities typically live in a `manage_menu`-style
domain — try it directly first, and only escalate to `search_capabilities`/`get_available_domains` on
an `entity_set_not_found`/`domain_not_found`-style rejection rather than re-discovering a domain that
already resolved earlier this session. This skill is connector-agnostic: it names entities, tasks, and
fields, not any one connector's literal tool names.

## Golden rule: don't decide the structure silently — ask

The sections below give a real decision framework, but "which menu," "which group," and "new vs.
existing" are product decisions about what a user will look for and where — wrong guesses cost
findability and muscle memory, and are annoying to unwind later. **Whenever the framework below doesn't
make the answer obvious, stop and ask the user** which existing menu/group to extend, or confirm they
actually want a new one — do not silently pick the conservative-sounding option and proceed. This
applies especially to:

- Whether a new screen/report/task belongs on an **existing menu** or needs a **new one**.
- Whether it belongs in an **existing group** or needs a **new group**.
- **Which menu type (list bar / tile / tree) a brand-new menu should use.** Even when the profiles
  below point clearly to one type, state the recommendation and *confirm it with the user before
  creating the menu* — don't just proceed on your own read of the fit. This is a one-way door in
  practice: the three types aren't a config flag you flip later, they're a different structure and a
  different modeling experience, so get it confirmed before, not after, groups and items pile up on
  top of it.
- Any **delete** (`task_delete_menu`/`task_delete_list_bar_grp`/`task_delete_tile_grp`/etc.) or
  **role-grant change** that removes access someone currently has — confirm intent first, these are
  easy to get wrong quietly.

Only skip asking when the answer is genuinely unambiguous — e.g. the user names the exact group, or
there is exactly one menu of the relevant platform/type and the request obviously belongs in an
existing group already covering that subject.

### Step 0: propose a plan

Don't confirm these decisions one at a time as they come up mid-build — that produces a series of
disconnected yes/no prompts and lets an earlier answer quietly commit you before the later ones are
even asked. Before making the first `stage_resource`/`stage_task` call, work through
`references/menu_design_guide.md`'s "Recommended design workflow" (its steps 1-6: personas/goals,
legitimate starting points, organizing principle, clustering/naming, menu type, new-vs-existing menu)
against the actual request, and turn the result into **one written plan** that bundles every decision
at once — menu, group, item(s), type, naming, `order_no`, and role visibility. Present that whole plan
to the user and get their explicit confirmation on it as a package before creating anything. Only
after that confirmation move on to the "Creating things" mechanics below. If the user's answer changes
one part of the plan, re-confirm the updated plan as a whole rather than resuming piecemeal. This is
the `thinkwise_software_factory_mcp_base` "Confirm-before-mutate" convention applied at this skill's
own grain; it isn't superseded by anything below about these entities' write mechanics.

## What belongs in the menu at all

Before deciding *which* menu/group/type, decide whether the object is a legitimate starting point in
the first place — the menu is a curated set of places to begin, not an inventory of every model
object. Add a subject/task/report only when a real user starts or resumes work from it directly;
route anything that acts on a *selected* record (a contextual task, a record-specific report, a
child/detail table) onto that subject instead, not the menu.

Quick smell test — lean toward **not** a menu item when it's:
- a link/child/history/staging table naturally reached through a master-detail relationship,
- a task or report that requires a current record to make sense,
- a one-off migration/diagnostic/seed-data utility,
- a near-duplicate of something already reachable elsewhere with no distinct audience benefit.

For the full in/out criteria, organizing-principle choice, naming, ordering, group-size heuristics,
contextual-navigation alternatives, and anti-patterns, see
`references/menu_design_guide.md` — that's the design layer this section only summarizes; the rest of
this file covers how to wire whatever you land on through the API.

## The three levels

| Level | Entities | Notes |
|---|---|---|
| **Menu** | `menu` | Keyed by `(model_id, branch_id, menu_id)`. `menu_type` (enum: `list_bar`=0, `tree`=1, `tile`=2) and `menu_platform` (bitmask, see below) are set once and define the whole container. |
| **Group** | `list_bar_grp` / `tile_grp` / `module_grp` | A labelled section heading. List bar and tile groups are **flat — no nesting**. `module_grp` alone has a self-referencing `grp_module_grp_id`, allowing true multi-level folders — see "Which type" below for why that's a narrow upside. |
| **Item** | `list_bar_item` / `tile` | Points at exactly one target via `menu_item_type` (enum: `tab`=0, `report`=2, `task`=3 — value `1` is `deprecated`) plus the matching `tab_id`/`tab_variant_id`, `report_id`/`report_variant_id`, or `task_id`/`task_variant_id`. |

## Which type to use

All three types can technically hold the same tabs, reports, and tasks — what separates them is the
audience, the item count, and (verified below) how well the API actually supports maintaining them.

| Type | Verdict | Use it for | Why |
|---|---|---|---|
| **List bar** (`list_bar`) | **Default** | Internal, back-office applications — several groups, each holding several-to-many items. | Dense, flat, well-supported API (see task reference below). Start here unless one of the others has a specific reason to win. |
| **Tile** (`tile`) | **Situational** | A small, curated set of entry points (roughly a dozen, not dozens); customer-facing portals/self-service where the audience isn't trained on the app; handheld/touch/kiosk; a landing "front door" before a list-bar back office. | `tile_size` (enum: `small`/`medium`/`wide`/`large`) gives visual prominence, but tiles don't compress — past a handful you're scrolling a wall of squares. |
| **Tree** (`tree`) | **Avoid** | Only a subject with genuine multi-level hierarchy that list bar/tile's flat groups truly can't express, and even then treat it as a last resort. | Legacy from the Windows/Web GUIs; Thinkwise has said Universal UI is bringing list bar "groups inside parent groups" nesting, closing tree's one advantage. **Also a real API gap, verified live in this domain**: `module_grp` has zero bound tasks (no create/rename/delete helpers — list bar and tile each have full sets, see below), and there is no `module_item`-equivalent entity exposed in this domain at all — tree's individual menu items aren't reachable through this API the way list bar/tile items are. If a tree menu already exists, treat it as migration debt toward list bar, not a foundation to keep building on. |

## New menu, or the existing one?

A `menu` is keyed to a **type + platform** combination, not to a role or a workflow.

**Create a new menu when** a genuinely new `menu_platform` target has no menu yet, or a truly distinct
application in the same model needs its own top-level entry point (rare — usually a module/role
concern, not a menu one).

**Stay on the existing menu when** the real need is per-role visibility (use grants — see below) or a
different look on a different device (there is no built-in responsive/conditional menu switching;
forking the menu just gives two menus to keep in sync). As of Thinkwise Platform 2026.1, Universal UI
is the only supported interface — treat `universal` (bit `8`) as the one menu type actively maintained
going forward; new work shouldn't default to spreading itself across legacy `windows`/`web`/`mobile`
menus.

**Whenever this leads to actually creating a new menu, verify the `menu_type` choice with the user
first** — state which of list bar / tile / tree the "Which type to use" table above points to and why,
and get their confirmation before creating it, even if the fit looks obvious. See the golden rule above.

`menu_platform` is a bitmask — one menu can cover several platforms at once:

| Platform | Value |
|---|---|
| `windows` | 1 *(legacy)* |
| `web` | 2 *(legacy)* |
| `mobile` | 4 *(legacy)* |
| `universal` | 8 *(current)* |

Combinations add (`windows_web`=3, `windows_web_mobile_universal`=15, etc.) — verified live, e.g. a
real `customer_portal_tile` menu uses `menu_platform=10` (`mobile_universal`).

## New group, or an existing one?

List bar and tile groups are flat — a group name is a promise about what a user finds under it.

**Create a new group when** the items are a distinct business subject, workflow stage, or audience a
user would look for under its own label (a real model separates `CRM`, `HR`, `projects`, `finance`,
`control`, `settings` as six sibling groups on one list-bar menu — each a clearly different subject,
not a table-technical split).

**Add to an existing group when** the item is another variant of something already there, or a
follow-on step in the same workflow the group already represents.

**Don't fake nesting** with label prefixes ("Sales – Invoicing", "Sales – Reporting") — list bar/tile
groups can't nest, and it reads as clutter, not structure. A subject with genuine two-level hierarchy is
the one case `module_grp`'s nested groups earn their keep, weighed against tree's API limitations above.

## Placing items

- **One item per target** — don't overload one item for two purposes; add a second item instead.
- **Order by workflow, not alphabet** — see `order_no` below.
- **Reach for menu search before more structure.** Once a menu holds many items, turn on
  `menu.show_filter` rather than inventing another layer of grouping to compensate.
- **When more than one existing group is a plausible fit, ask — don't silently pick the "closest"
  one.** The same goes for a non-obvious placement within the group (order relative to neighbors) or
  icon choice: state the option you'd lean toward and why, and get the user's confirmation, rather than
  resolving the ambiguity on your own judgment.

## Security: grant, don't fork

Every level carries its own per-role overview entity, each with a `granted` flag and a `rights_icon`:

| Level | Overview entity | Key |
|---|---|---|
| Menu | `role_menu_overview` | `(model_id, branch_id, role_id, menu_id)` |
| List bar group | `role_list_bar_grp_overview` | `(…, menu_id, list_bar_grp_id)` |
| List bar item | `role_list_bar_item_overview` | `(…, list_bar_grp_id, list_bar_item_id)` |
| Tile group | `role_tile_grp_overview` | `(…, menu_id, tile_grp_id)` |
| Tile | `role_tile_overview` | `(…, tile_grp_id, tile_id)` |

`rights_icon` enum: `super_user`=1, `grant`=2, `read`=3, `hidden`=4, `unauthorized`=5.

**Verified live**: a row already exists for **every role** at every level (e.g. querying
`role_menu_overview` for one menu returned one row per role in the model, each already `granted: true`
or `false`). This is the same "overview" pattern seen elsewhere in the platform (an implicit row per
candidate combination) — **locate the existing row by its full key and edit `granted`; don't try to add
one.** Group/item overview rows also carry an `available` flag, distinct from `granted` — treat
`available` as whether this role's group is even eligible to be configured here, and `granted` as the
actual on/off switch you're setting. `rights_icon` reads as the resulting computed state rather than a
field to set directly — patch `granted` and re-read `rights_icon` to confirm the effect, rather than
writing to `rights_icon` itself.

**This is why "different roles need different menus" is almost never a real reason to create a second
menu** — set the grant at whichever level (menu/group/item) matches what should actually be hidden.

## Creating things: verified entity/task reference

### Menu

No bound "create" task exists for `menu` in this domain — create it as a plain new record (the generic
connector's insert/`add` flow) with `menu_id`, `menu_type`, `menu_platform` set, then `show_filter`/
`show_open_documents`, and `icon_id` (a suitable icon per `thinkwise_software_factory_icons` — broad
and stable, representing the whole application/module the menu covers, not one frequently-used screen)
as wanted. `task_rename_menu` (params `branch_id`, `from_menu_id`, `to_menu_id`)
and `task_copy_menu` (params `from_menu_id`, `to_menu_id` — clones an existing menu's whole group/item
structure, a good starting point for a new platform variant of one you already have) and
`task_delete_menu` (params `branch_id`, `menu_id`) round out the lifecycle.

### Groups

| Entity | Create | Mandatory params | Then edit directly |
|---|---|---|---|
| `list_bar_grp` | `task_create_list_bar_grp`, bound to `list_bar_tree` | `list_bar_grp_id` (optional `icon_id`) | `list_bar_grp_description`, `order_no` |
| `tile_grp` | `task_create_tile_grp`, bound to `tile_tree` | `tile_grp_id` | `tile_grp_description`, `order_no`, `icon_id` |
| `module_grp` | *(no bound task — plain add)* | — | all fields, including `grp_module_grp_id` for nesting, and `icon_id` |

All three group types carry an `icon_id` — set one deliberately as part of creating the group (the
shared business category it represents, e.g. Sales/Warehouse/Finance), per
`thinkwise_software_factory_icons`, rather than leaving it blank because the field itself is optional.

`list_bar_tree`/`tile_tree` are read-oriented "design tree" helper entities mirroring the Software
Factory's own tree view — every group is a root row in it (`parent_list_bar_grp_item_id` blank,
verified live), every item a child row under its group's `pk_col`. The create-group/create-item bound
tasks are bound to a row in this tree, addressed by its synthetic `pk_col`
(`{model_id}/{branch_id}/{menu_id}/…`) — bind to **any** existing row in the target menu (a group or an
item both work; the task creates the new group at the root regardless of which node you bound to, since
list bar/tile groups can't nest under one another anyway). **For the very first group in a brand-new,
completely empty menu** (no `list_bar_tree`/`tile_tree` rows exist yet to bind to), a plain add directly
on `list_bar_grp`/`tile_grp` is the fallback — **verified live**: staging it as a dependent record
under its parent `menu` (rather than a bare top-level insert) succeeds and correctly pre-populates the
key fields. The same dependent-record-under-parent approach is also the reliable way to bootstrap the
first *item* in a group — see Items below, where it's actually the recommended path generally, not
just for bootstrapping.

### Items

| Entity | Create | Mandatory params | Then edit directly |
|---|---|---|---|
| `list_bar_item` | `task_create_list_bar_item`, bound to `list_bar_tree` | *(none)* | `list_bar_item_description`, `menu_item_type`, `tab_id`/`tab_variant_id` or `report_id`/`report_variant_id` or `task_id`/`task_variant_id`, `order_no` |
| `tile` | `task_create_tile`, bound to `tile_tree` | *(none)* | same fields, plus `tile_size` |

**Verified, easy to miss**: both create-item tasks take **zero parameters** — they create a shell row
with a system-generated key, unlike `list_bar_grp`'s creation which requires an ID up front.

**Verified live, and it contradicts the zero-parameter appearance above**: `task_create_list_bar_item`
stages with no settable fields at all, but committing it can then fail with a mandatory-field validation
error on `menu_item_type` — a field the task never exposes a way to set. Don't rely on this bound task
for creating items. Instead, create the item as a plain dependent-record add under its parent group (the
same "add under parent" approach used for the group bootstrap case above) — this correctly defaults
`menu_item_type`, and lets `list_bar_item_id`, the target field (`tab_id`/etc.), and `order_no` all be
set in the same staging session before committing. Treat `task_create_tile` as suspect for the identical
reason until it's actually been verified — it follows the same zero-parameter creation pattern.
**Also confirmed live** — another instance of the general "a multi-field write can silently drop one
field" behavior in `thinkwise_datamodeling_guidelines`'s quirks section: `list_bar_item_description`
(and the same thing separately confirmed on `list_bar_grp_description`) can silently revert to `null`
after a *later, otherwise-successful* patch in the same staging session (e.g. right after setting
`list_bar_item_id`/`task_id`) — re-check the description field's value in the response after any
subsequent patch and re-set it if it reverted, rather than assuming a single earlier patch call
permanently stuck. **Re-tested and fixed, verified live**: setting the description *last* — after
`list_bar_item_id`/`tab_id` (or `list_bar_grp_id`, for a group) — in one combined `stage_resource` call
avoided the drop entirely; the description held correctly with no follow-up patch needed. Order the
properties this way rather than defaulting to a separate corrective patch.

There is no rename task for `list_bar_item`/`tile` — that's fine, because the generated key isn't
shown to end users; only `list_bar_item_description`/`tile_description` is. **Also verified live**:
leaving that description blank is a legitimate, common choice — the design tree's own `display_value`
falls back to the linked object's name plus a type suffix (e.g. `customer (table)`) when no override
description is set, matching what a real production menu does across every item inspected.

**Verified live, silent-failure trap**: setting `tab_id`/`report_id`/`task_id` (or any lookup field) by
display text can silently fail — no error raised, the field simply stays unset — when the target's
display text carries extra wording beyond its plain identifier (e.g. a table described as
`"Employee schedule (Scheduler subject)"` whose actual id is `employee_schedule`). Always re-check the
field's value in the response immediately after setting it; if it didn't take, force a literal/data-value
match on the identifier instead of relying on display-text resolution.

### Ordering

`order_no` follows the platform's usual 10/20/30… convention with gaps for later insertions (verified
live on both groups and items). **Prefer editing `order_no` directly over the `task_move_*`/
`task_renumber_*` bound tasks** (`task_move_list_bar_item_order_no`, `task_move_list_bar_grp_item`,
`task_renumber_list_bar_grp`, and their tile equivalents) — verified live, every one of these takes
**zero parameters**, meaning they carry no machine-usable direction/target and are effectively
drag-and-drop artifacts from the designer canvas, not a scriptable reordering API. Resize
(`task_resize_tile_small`/`_medium`/`_wide`/`_large`) is the one exception worth using directly since
the target tile is fully identified by the bound key — but editing `tile.tile_size` directly is simpler
and equally valid, since it's a plain editable field.

### Moving an item to a different group

**Reassigning an item's group is not a field edit** — `list_bar_grp_id` (or `tile_grp_id`) is part of
the item's composite key, so there is no in-place "move" write. The working sequence: delete the item
from its current group (`task_delete_list_bar_item`/`task_delete_tile`), then add it fresh as a
dependent record under the new parent group (the same approach used for creating an item at all — see
Items above), reusing the **same** `list_bar_item_id`/`tile_id`, `tab_id`/`report_id`/`task_id`, and
`order_no` for continuity. This applies whenever reorganizing existing items into a new or different
group — e.g. consolidating reference/lookup tables scattered across several subject groups into one
dedicated group.

### Translating new menus, groups, and items

`menu`, `list_bar_grp`/`tile_grp`/`module_grp`, and `list_bar_item`/`tile` are all translatable
objects. Bound-task creation typically leaves a bracket-placeholder label to translate, but the
dependent-record-add workaround used above sometimes leaves no `transl_object` at all, and a group's
real translation type isn't what its name suggests — see `references/menu_translation_notes.md` for
the verified specifics, the backfill recipe, and the final translation-completeness check.

### Cross-reference (read-only)

`menu_tab` / `menu_report` / `menu_task` list which tabs/reports/tasks are referenced by which menus
across the whole model — query these before renaming or removing a table/report/task to see every menu
item that would break.

## Bound-task quick reference

| Entity | Bound tasks |
|---|---|
| `menu` | `task_copy_menu`, `task_rename_menu`, `task_delete_menu`, `task_menu_mark_new_object_approved`/`_disapproved`, `task_show_history`, `task_unlink_generated_object` |
| `list_bar_tree` | `task_create_list_bar_grp`, `task_create_list_bar_item`, `task_move_list_bar_grp_item`, `task_go_to_list_bar_tree_item` |
| `list_bar_grp` | `task_delete_list_bar_grp`, `task_rename_list_bar_grp`, `task_renumber_list_bar_grp`, `task_create_all_tables_list_bar_grp` (bulk-populate an *existing* group with one item per table — bound to `list_bar_grp`, so it can't bootstrap a brand-new menu's first group), `task_list_bar_grp_mark_new_object_approved`/`_disapproved`, `task_show_history`, `task_unlink_generated_object` |
| `list_bar_item` | `task_delete_list_bar_item`, `task_move_list_bar_item_order_no`, `task_from_object_to_menu_modeler_list_bar` (UI navigation helper — zero params, not for adding items), `task_list_bar_item_mark_new_object_approved`/`_disapproved`, `task_show_history`, `task_unlink_generated_object` |
| `tile_tree` | `task_create_tile_grp`, `task_create_tile`, `task_move_tile_grp_item`, `task_resize_tile_small`/`_medium`/`_wide`/`_large`, `task_go_to_tile_tree_item` |
| `tile_grp` | `task_delete_tile_grp`, `task_rename_tile_grp`, `task_renumber_tile_grp`, `task_move_tile_grp_order_no`, `task_tile_grp_mark_new_object_approved`/`_disapproved`, `task_show_history`, `task_unlink_generated_object` |
| `tile` | `task_delete_tile`, `task_move_tile_order_no`, `task_renumber_tile`, `task_from_object_to_menu_modeler_tile`, `task_tile_mark_new_object_approved`/`_disapproved`, `task_show_history`, `task_unlink_generated_object` |
| `module_grp` / `module_tree` | *(none — zero bound tasks; plain add/edit/delete only)* |

## Pre-flight checklist

- **Ask before deciding structure.** New menu vs. existing, new group vs. existing — if the framework
  above doesn't make it obvious, ask the user rather than guessing.
- **Always verify the menu type before creating a new menu** — state the recommended `menu_type`
  (list bar / tile / tree) and why, and get the user's confirmation first, even when the fit seems
  obvious.
- Confirm the connector's actual domain key for menus (`manage_menu`-style) before assuming
  `sf/manage_menu` holds everywhere.
- Set `icon_id` on every new menu/group (menu, list bar/tile/module group) to a suitable icon per
  `thinkwise_software_factory_icons` — don't leave it blank by default. A menu item never needs its own
  icon; it always shows the referenced table/report/task's icon.
- Default to **list bar**; reach for **tile** only for small item counts or customer-facing/touch
  contexts; treat **tree** as something to migrate away from, not build on — it has real, verified API
  gaps (no bound tasks on `module_grp`, no exposed item entity) on top of the UX case against it.
- The bound create-item tasks (`task_create_list_bar_item`/`task_create_tile`) can fail on commit with
  a mandatory-field error on `menu_item_type` that they expose no way to set — create items as a
  dependent-record add under the parent group instead; it works reliably and sets every field
  (description, target, order) in one step. Creating a `list_bar_grp`/`tile_grp` **does** require an ID
  up front.
- When setting `tab_id`/`report_id`/`task_id` (or any lookup) by display text, verify the field actually
  populated — display text with extra wording beyond the plain identifier can silently fail to resolve,
  with no error raised.
- Don't rely on `task_move_*`/`task_renumber_*` for scripted reordering — they're zero-parameter
  drag-drop artifacts. Edit `order_no` directly instead.
- To hide/show something per role, **locate the existing `role_*_overview` row and edit `granted`** —
  never try to add a new one; a row already exists for every role at every level.
- Query `menu_tab`/`menu_report`/`menu_task` before renaming or removing a table/report/task referenced
  by any menu.
- Menu/group/item objects created through a proper bound task get a bracket-placeholder translation
  (e.g. `[main_menu]`), not a finished label — follow `thinkwise_software_factory_translation_objects`
  to fill these in once the structure is built. **But a `list_bar_grp`/`list_bar_item` created via the
  dependent-record-add workaround gets no `transl_object` at all** — this is recoverable via the API
  (manually add `transl_object`, then run the generic `task_generate_transl_objects` bound to it — see
  `thinkwise_software_factory_translation_objects`), not a gap only fixable manually in the Software
  Factory's own UI.
- The brand-new-empty-menu bootstrap (first group, and first item in a group) is verified working via a
  dependent-record add under the parent (`menu` for the first group, `list_bar_grp`/`tile_grp` for the
  first item) rather than a bare top-level insert.
