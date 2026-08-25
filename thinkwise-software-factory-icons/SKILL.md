---
name: thinkwise-software-factory-icons
description: Reference guide for the icon repository (icon/icon_grp/icon_tag) and every verified place an icon can be assigned in a Thinkwise Software Factory model — tables, tasks, reports, prefilters, menus, cube views, domain elements, message options, form/report navigation, conditional layouts — plus the semantic vocabulary for choosing the right icon per object/action/state. Use whenever an MCP connector with Software Factory access creates, inspects, reuses, replaces, or assigns an icon, or before calling get_entity_definition/execute_odata_query/stage_resource/stage_task against icon, icon_grp, icon_tag, or any `*icon_id`/`image_id` field.
---

# Icons in the Thinkwise Software Factory

Every entity, field, and bound task below was confirmed live against a real connected model (domain
`sf/software_development`) — not guessed from documentation. The semantic guidance (which concept fits
which object/action/state) is design guidance, not a database rule; it lives in
`references/icon_selection_guide.md` to keep this file focused on mechanics. Follow
`thinkwise_software_factory_mcp_base` for connector/model/branch resolution and the shared
confirm-before-mutate / ask-don't-default rules before making any of the writes described here.

## The repository: `icon` / `icon_grp` / `icon_tag`

| Entity | Key | Purpose |
|---|---|---|
| `icon` | `(model_id, branch_id, icon_id)` | One uploaded image. `icon` (`SQLSERVER_SF.FileContract`) and `icon_data` (`Edm.Binary`) carry the file; `icon_title` and `icon_extension` are separate scalar fields; `icon_display` is a read-only computed `"{icon_title}.{icon_extension}"` (verified live, e.g. `icon_title="snacks-icon"`, `icon_extension="svg"` → `icon_display="snacks-icon.svg"`). `icon_grp_id` is optional — a live sample across several real models showed most icons left it blank; grouping is opt-in, not enforced. `order_no_grp_or_item` sorts within a group. |
| `icon_grp` | `(model_id, branch_id, icon_grp_id)` | A flat label for organizing the repository (`icon_grp_description`, `order_no`). No nesting — same flat shape as `list_bar_grp`/`tile_grp`. |
| `icon_tag` | `(model_id, branch_id, icon_id, tag_id)` | The platform's generic tagging mechanism (`tag` entity) applied to an icon, plus a free `value`. A second, orthogonal way to classify an icon beyond `icon_grp_id` — e.g. tag by visual family/source without forcing every icon into one group. |

Both `icon` and `icon_grp` are branch-scoped like every other design-time object (`model_id`/`branch_id`
in the key) — see `thinkwise_software_factory_mcp_base` on why an unfiltered query can silently mix rows
from an unrelated model/branch.

### Bound tasks on `icon` (verified)

| Task | Parameters | Effect |
|---|---|---|
| `task_rename_icon_name` | `icon_id`, **and all of** `from_icon_name`, `from_icon_name_no_extension`, `from_icon_name_extension`, `to_icon_name`, `to_icon_name_extension` (every one mandatory) | Renames the icon's filename. Supply the icon's *current* title/extension exactly, not just the target — this isn't a simple "new name" call. |
| `task_enrichment_decolorize_svg` | bound by `icon_id`, but its one real parameter is `icon_title` (mandatory) — re-supply the icon's current title | Strips embedded fill color from an SVG so Universal UI's theme-based recoloring can work (light/dark). Only meaningful for monochrome SVG concepts, not photographic/multicolor assets. |
| `task_update_icon_usage` | `from_icon_id` (bound) + `upload_new_icon` (bool) + either `to_existing_icon_id` **or** `to_new_icon`/`to_new_icon_data` | The consolidation tool: redirects **every** existing assignment of `from_icon_id` across the whole model to a different existing icon (dedup a duplicate) or to a freshly uploaded replacement — in one step, instead of hand-editing every `*_icon_id` field that pointed at it. |
| `task_download_selected_icon` | `icon`, `icon_data` | Downloads a copy of the icon. |
| `task_get_icon_as_varbinary_literal` | `icon_id` → `varbinary_literal` | Produces a SQL varbinary literal for the icon's bytes — useful for seeding icons via a control procedure/script rather than the API's own upload path (see below). |
| `task_delete_icon` | `icon_id` | Deletes the icon record. Check the assignment points below first — nothing here blocks the delete just because the icon is still referenced. |

### Bound tasks on `icon_grp`

`task_rename_icon_grp` (`from_icon_grp_id`/`to_icon_grp_id`) and `task_delete_icon_grp` — **no create
task exists**; add a new `icon_grp` as a plain record (`icon_grp_id`, `icon_grp_description`, `order_no`),
the same pattern used for `module_grp` elsewhere in the platform.

### Uploading a brand-new icon — unverified through this API

`icon.icon` is typed `SQLSERVER_SF.FileContract`, a file-upload field distinct from the plain
`icon_data` binary column, and there is no bound "upload" task — a new icon looks like a plain add
setting `icon`/`icon_data`/`icon_title`/`icon_extension` together. **This mechanic has not been
exercised live in this session** — whether an MCP connector's staging flow can actually carry a
`FileContract` payload (vs. only reading/renaming/reassigning existing icons) is unconfirmed. Until
verified, treat *uploading a new file* as something to do in the Software Factory's own Icons
repository UI, and use the API confidently for everything else here: reading, grouping, tagging,
renaming, decolorizing, consolidating (`task_update_icon_usage`), and assigning an existing `icon_id`
to any of the fields below.

## Every verified assignment point

Confirmed by pulling the full property list of every candidate entity against the live domain (not just
`icon`'s own reverse navigation, which misses one real case below) — this is the complete set of
entities that carry an icon-referencing field, not a curated subset.

**Almost every one of them carries a pair, not a single field**: an `<x>icon_id` (or `image_id` on
`cube_view_field_conditional_layout`) that's a foreign key into the shared `icon` repository, *and* its
own local `<x>icon`/`<x>icon_data` (`SQLSERVER_SF.FileContract` + `Edm.Binary`) that uploads a file
straight onto that row, bypassing the repository entirely. Both are always present or absent together,
with **one verified exception**: `tab_task` has *only* the local `icon`/`icon_data` pair — a
table-specific task icon override can never be picked from the shared repository, only uploaded fresh
onto that one `tab_task` row.

**Prefer the `_id` repository field whenever both exist.** The local `icon`/`icon_data` pair is a
parallel, older upload mechanism — it works, but an icon set that way doesn't show up in the shared
repository for reuse, can't be redirected via `task_update_icon_usage`, and won't benefit from
`task_enrichment_decolorize_svg` being run once and shared everywhere. Reach for it only where there is
no `_id` alternative (`tab_task`) or the user explicitly wants a one-off asset kept out of the shared
repository.

| Object | Repository field | Local-upload field(s) | What it controls |
|---|---|---|---|
| `branch` | `icon_id` | `icon`/`icon_data` | The branch's own icon (design-time — shown wherever a branch is picked/listed, not necessarily an end-user surface). |
| `tab` | `icon_id` | `icon`/`icon_data` | A table/view's icon — its document, menu item, and detail navigation. |
| `tab_variant_overview` | `icon_id` | `icon`/`icon_data` | Per-variant override of a table's icon. |
| `task` | `icon_id` | `icon`/`icon_data` | A task's icon — action bars, menus, wherever the task appears. |
| `task_variant_overview` | `icon_id` | `icon`/`icon_data` | Per-variant override of a task's icon. |
| `tab_task` | *(none)* | `icon`/`icon_data` only | A table-specific override of a task's icon for one table's action bar — **upload-only, no repository option**, verified live. |
| `report` | `icon_id` | `icon`/`icon_data` | A report's icon. |
| `report_variant_overview` | `icon_id` | `icon`/`icon_data` | Per-variant override of a report's icon. |
| `tab_prefilter` | `icon_id` | `icon`/`icon_data` | A single prefilter's icon. |
| `tab_prefilter_grp` | `icon_id` | `icon`/`icon_data` | A prefilter group's icon. |
| `menu` | `icon_id` | `icon`/`icon_data` | The whole menu's icon (shown when switching between menus). |
| `list_bar_grp` | `icon_id` | `icon`/`icon_data` | A list-bar menu group's icon. Weak entity — its edit key is `(model_id, branch_id, menu_id, list_bar_grp_id)`, not `list_bar_grp_id` alone; see `thinkwise_software_factory_mcp_base` on weak-entity edit keys. |
| `tile_grp` | `icon_id` | `icon`/`icon_data` | A tile group's icon. |
| `module_grp` | `icon_id` | `icon`/`icon_data` | A legacy tree menu's module-group icon. |
| `tab_task_grp` | `icon_id` | `icon`/`icon_data` | A table action-bar task-group icon. |
| `tab_report_grp` | `icon_id` | `icon`/`icon_data` | A table action-bar report-group icon. |
| `cube_view` | `cube_view_icon_id` | `cube_view_icon`/`cube_view_icon_data` | A cube view's icon (e.g. on a BI menu/tile). |
| `msg_option` | `icon_id` | `icon`/`icon_data` | A modeled message's option button (confirm/retry/cancel-style choices) — see `thinkwise_software_factory_messages`. |
| `elemnt` | `icon_id` | `icon`/`icon_data` | A single domain element's (enum value's) icon — renders when the owning `dom.control_id` is set to an icon-based control (an image/icon combo or radio-style presentation), not for a plain text/combo domain. |
| `cube_view_field_conditional_layout` | `image_id` (confirmed FK into `icon.icon_look_up`, same repository, just named "image") | `image`/`image_data` | An icon-based conditional layout — showing an icon instead of/alongside color for a computed condition. See `thinkwise_software_factory_conditional_layouts` and `thinkwise_software_factory_cubes`. |
| `screen_component` | `tab_page_icon_id` | `tab_page_icon`/`tab_page_icon_data` | A custom screen component's tab-page icon (multi-page custom components). `screen_component_type_icon`/`_data` also exists but is the component *type's* own icon, not settable per instance. |
| `col` | `form_next_grp_icon_id`, `next_tab_icon_id` | matching `form_next_grp_icon`/`_data`, `next_tab_icon`/`_data` | A column's "jump to related form group" / "jump to related tab" navigation icons. |
| `report_parmtr` | same two `_id` fields | same local pairs | Same navigation-icon pattern, on a report parameter. |
| `task_parmtr` | same two `_id` fields | same local pairs | Same navigation-icon pattern, on a task parameter. |
| `tab_variant_form_overview` | same two `_id` fields | same local pairs | Per-variant override of a table column's navigation icons. |
| `report_variant_form_overview` | same two `_id` fields | same local pairs | Per-variant override of a report parameter's navigation icons. |
| `task_variant_form_overview` | same two `_id` fields | same local pairs | Per-variant override of a task parameter's navigation icons. |

Notably **absent**: `tile` and `list_bar_item` carry no icon field of their own at all — a menu item
always shows the icon of whatever it points to (`tab`/`report`/`task`, or that object's variant
override), confirming the platform's own "menu item icon = referenced object's icon" convention rather
than a separate menu-only vocabulary. `action_bar` likewise carries no icon field — per-action display
(icon-only vs. icon+text vs. overflow) is controlled by `action_bar`'s own `default_display_type` enum
(`icon_text_*`/`text_only_*`/`icon_only_*`/`overflow`/`hidden` values, verified live), not by a separate
icon field. `task_button`/`report_button` were checked directly and don't exist as entity sets in this
domain at all.

## Assigning or changing an icon

Whenever a skill or task involves creating or configuring any object in the table above (a table, a
task, a report, a prefilter, a menu/group, a cube view, a domain element on an icon-capable domain, a
message option, a form/report navigation group, an icon-based conditional layout), **assigning it a
suitable icon is part of finishing that object, not an optional extra** — don't leave the field unset
by default just because the field itself isn't mandatory — **unless the model's existing convention for
that exact object type says otherwise (step 0 below).**

0. **Follow the model's existing convention for this object type — don't impose one.** Before assigning
   an icon to a *newly created* object, check how existing rows of that same entity (e.g. other
   `list_bar_grp` rows on the same or other menus, other `menu` rows, other `tab_prefilter_grp` rows)
   actually use the field: count how many have the `_id`/local-upload field set vs. null.
   - If existing rows overwhelmingly **do** use icons, assign one to the new object too (proceed with
     steps 1+ below).
   - If existing rows overwhelmingly **don't** use icons (e.g. a model has several `list_bar_grp` rows
     and none of them carry an icon), **don't** add one to the new object either — match the model's
     established look rather than making one object inconsistent with its peers. Say so explicitly
     ("this model's existing menu groups don't use icons, so I'm leaving this one unset to match") rather
     than silently skipping it.
   - If usage is mixed with no clear majority, or there are no existing rows of that type to compare
     against (this is the first one), fall back to the default in the paragraph above: assign an icon.
   - Judge this **per object type**, not globally — a model can consistently iconify `tab` rows while
     never iconifying `list_bar_grp` rows; don't let one type's convention decide another's.
1. **Reuse before uploading.** Query `icon` (optionally filtered by `icon_grp_id`/`icon_tag`) for an
   existing icon matching the concept before creating a new one — see
   `references/icon_selection_guide.md` for how to judge "matching."
2. **If nothing in the repository fits, don't upload or pick unilaterally — ask the user.** Present
   what you searched for and why nothing matched, then ask (via `AskUserQuestion` or a plain question)
   whether they want to (a) upload a specific icon file themselves for you to add to the repository, or
   (b) name a specific existing icon (even an imperfect fit) to use instead. This mirrors
   `thinkwise_software_factory_maps_component`'s existing rule for `elemnt.icon` — don't silently pick a
   "close enough" icon or leave the field blank when no confident match exists.
3. Set the target field to the chosen `icon_id` (repository) via `patch_resource`/`stage_resource` —
   prefer the `_id` field over the entity's own local `icon`/`icon_data` upload pair wherever both exist
   (see "Prefer the `_id` repository field" above); use the local pair only on `tab_task`, where it's the
   only option, or when the user explicitly wants a one-off file kept out of the shared repository.
4. **Before changing an icon that's already assigned elsewhere**, check whether other objects share it —
   an unrelated screen can shift visually if you edit or replace a widely-reused icon. Prefer
   `task_update_icon_usage` (redirect *this* object's assignment to a different, more specific icon by
   changing the field directly; use the task only when you actually want to repoint *every* usage of one
   icon at once, e.g. consolidating duplicates).
5. Per `thinkwise_software_factory_mcp_base`'s confirm-before-mutate rule: state which icon (existing or
   new) you're assigning and why before the first mutating call, especially for a menu/table/task icon
   that's user-visible across the whole application, not just a low-stakes internal object.
6. **After setting `dom.control_id` to an icon-capable control** (e.g. `IMAGE_COMBO`) so `elemnt.icon_id`
   will actually render: if the domain's underlying data type is numeric (`SMALLINT`/tinyint/etc.), the
   Software Factory silently resets `dom.alignment` to `right` (1) as a side effect of the control_id
   change — even though a left-aligned icon/enum-style presentation is what you actually want. Confirmed
   live: setting `control_id=IMAGE_COMBO` on 4 numeric domains flipped all 4 to `alignment=1`, while
   sibling element-bearing domains in the same model that hadn't had their control_id touched stayed at
   `alignment=0` (left). Re-read `dom.alignment` right after any `control_id` write on a numeric domain
   and patch it back to `left` (numeric value `0` — the string key `"left"` is rejected with
   `invalid_input`) unless the model's own convention for that domain type is actually right-aligned.

## Auditing a model for missing icons

When asked to find every icon field that's currently unset — rather than assigning icons for objects
already being built — don't treat every null `*icon_id`/`image_id` as a gap to fill. Several of the
fields in the table above are optional per-row overrides for a feature that many models never turn on,
so they're null by design, not by omission:

- **Navigation icons** (`col`/`report_parmtr`/`task_parmtr`/`*_variant_form_overview`'s
  `form_next_grp_icon_id`/`next_tab_icon_id`) only render when that column/parameter actually has the
  matching jump-to-group/jump-to-tab navigation configured. If no row in the model uses that
  navigation feature at all, leave these alone.
- **`elemnt.icon_id`** only renders when the owning `dom.control_id` is an icon-capable control (see
  the pre-flight checklist below) — a domain element on a plain `COMBO`/text control never displays it.
- **`screen_component.tab_page_icon_id`** only matters for a multi-page custom screen component — a
  single-page component (or a model with no multi-page custom components at all) leaves it null
  meaningfully.
- **`tab_task`'s local `icon`/`icon_data` override** only matters where a table actually overrides one
  of its tasks' icons for its own action bar — most tables never do, so an empty value there isn't a
  gap.

Before flagging any of these as "missing," check whether the underlying feature is used anywhere in
the model at all (e.g. count rows where the relevant field is *set*, not just where it's null) — a
feature sitting at zero usage means there's nothing to iconify, not several fields to fill in.

## Pre-flight checklist

- Confirmed connector/model/branch per `thinkwise_software_factory_mcp_base`.
- For a *newly created* object, checked whether existing rows of that same entity commonly use icons
  before assigning (or skipping) one — matched the model's existing convention per object type rather
  than defaulting to always-on; only defaulted to assigning when usage was mixed or there was no
  existing row to compare against. See "Follow the model's existing convention" (step 0) above.
- Searched the existing `icon`/`icon_grp`/`icon_tag` repository for a matching concept before proposing
  a new upload.
- Picked the *specific* field for the object at hand from the table above — note several entities have
  **two** variant-scoped fields (base `*_overview` vs. the plain entity), the two navigation-icon fields
  (`form_next_grp_icon_id` vs. `next_tab_icon_id`) are visibly different UI hooks, and most objects have
  **both** a repository `_id` field and a local `icon`/`icon_data` upload pair — used the `_id` field
  unless the object is `tab_task` (upload-only) or the user wants a one-off asset.
- If no existing icon in the repository fit, asked the user to upload one or name a specific icon,
  rather than picking the closest match unilaterally or leaving the field unset.
- For a status/enum-driven icon, checked that the owning `dom.control_id` is actually an icon-capable
  control before setting `elemnt.icon_id` — on a plain text/dropdown domain the field would go unused.
- After setting `dom.control_id` on a numeric-typed domain, re-checked `dom.alignment` and reset it to
  `left` if the Software Factory silently flipped it to `right` (see step 6 above) — don't assume
  alignment is untouched by a control_id change.
- Read `references/icon_selection_guide.md`'s relevant section (action/object/status/menu/prefilter/
  report/form-group) before picking the concept, not just the mechanical field.
- Before replacing or heavily editing a shared icon, checked its other assignments; used
  `task_update_icon_usage` for a genuine one-to-many consolidation rather than hand-editing every field.
- New-file upload not yet verified via this API — routed actual file uploads through the Software
  Factory's own Icons repository UI unless/until confirmed otherwise.
- If auditing an existing model for missing icons (not authoring new objects), confirmed the
  underlying optional feature (nav icons, multi-page screen components, per-table task icon
  overrides) is actually used before treating a null field as a gap — see "Auditing a model for
  missing icons" above.
