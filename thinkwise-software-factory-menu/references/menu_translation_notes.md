# Translating new menus, groups, and items

`menu`, `list_bar_grp`/`tile_grp`/`module_grp`, and `list_bar_item`/`tile` are all translatable
objects, and an object created through a proper bound creation task typically renders with an
auto-generated bracket-placeholder label (`[main_menu]`, `[scheduler]`) until translated, the same
auto-fallback pattern covered in `thinkwise_software_factory_translation_objects`.

**Behavior on this is inconsistent across sessions — check first, don't assume either way.** One
session found that a `list_bar_grp`/`list_bar_item` created via the dependent-record-add pattern
recommended above (needed because the bound creation tasks are unreliable — see "Items" above) did
*not* get an auto-generated `transl_object` at all: querying for one immediately after creating and
committing the group/item returned zero rows, not even a bracket-placeholder. A later session, adding
a group to an already-populated menu the same way, got a real bracket-placeholder `transl_object` (at
`type_of_object = list_bar`, the group's correct type per the trap below) with no extra steps. The gap
may be specific to bootstrapping the very first group/item in a brand-new, completely empty menu
rather than a general property of the dependent-record-add pattern — that specific case remains
unverified either way. **Query for the `transl_object` first, before assuming it's missing or assuming
it exists** — only reach for the backfill recipe below once a bare-id query has actually confirmed
zero rows.

**If it genuinely is missing, this is recoverable through the API, not a gap only fixable manually in
the Software Factory's own UI** — a narrow, per-parent-type generate variant
(`generate_transl_objects_list_bar_grp_*`) fails on its own hidden mandatory `branch_id`, and
hand-adding `transl_object_transl` directly fails too (`appl_lang_id` is hidden-yet-mandatory on a
fresh add). But routing through the *generic* `task_generate_transl_objects` bound to a
manually-added `transl_object` row works — see `thinkwise_software_factory_translation_objects`'s
"Backfilling a missing `transl_object` by hand" section for the full three-step recipe, verified live
for exactly this object type. Follow that instead of falling back to a manual step.

For everything else, after building out a menu's structure, run
`thinkwise_software_factory_translation_objects`' detection query to catch every new group/item still
showing its raw ID in brackets, rather than assuming a sensible-looking `..._id` naming means it's
already readable to end users. **Before calling the menu work done, run the translation completeness
gate from `thinkwise_datamodeling_guidelines`'s "Translating new objects" section** as the final
mechanical check — don't rely on remembering which group/item got translated during the build.

**A group's real translation type is not what its name suggests, and an item's may not exist at
all — see `thinkwise_software_factory_translation_objects`'s `type_of_object` section for the full
finding.** In short: a `list_bar_grp`'s actual, UI-rendered label lives at `type_of_object = list_bar`
(12), not `list_bar_grp` (78) — translating the latter commits with no error but is inert. And
`list_bar_item` (79) has no working consumer found at all; a table/report-linked item just displays
the linked object's own translation, so don't spend time translating the item directly for that case.
