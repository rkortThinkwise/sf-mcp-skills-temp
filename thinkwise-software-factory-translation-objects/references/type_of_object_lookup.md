# `type_of_object` — one enum, ~150 values, spanning almost the whole model

`type_of_object` is not specific to tables/columns/domain elements — it is the same enum used to key
*every* translatable concept in the platform, and it grows across platform versions. **Always fetch
the current values live** — `get_entity_definition` (or the connector's equivalent metadata call)
against `transl_object`/`transl_object_transl`'s `type_of_object` property/domain — rather than
trusting any hardcoded list, including the small one below.

**Lookup table — what to query for a given kind of thing.** Built from a live enum fetch plus
targeted live record checks against `INSIGHTS`/`MAIN` (confidence noted per row) — still confirm
before relying on any of these in a different model, but treat this as the starting point instead
of guessing from the object's own name.

| You're translating... | Query `type_of_object` | Value | Confidence / trap |
|---|---|---|---|
| A table/view's own name (and its plural, same row) | `tab` | 0 | live-confirmed |
| A column's label | `col` | 1 | live-confirmed; shared across tables with the same `col_id` — see Naming below |
| A column/`task_parmtr`/`report_parmtr`'s `form_next_grp_label` / `next_tab_label` (group or collapsible-section header) | `tab_label`, **not** the parent's own type | 33 | live-confirmed trap — see below |
| A domain element (dropdown/enum value label) | `dom_elemnt` | 2 | live-confirmed |
| A task's own name | `task` | 11 | live-confirmed |
| A task parameter's label | `task_parmtr` | 32 | live-confirmed |
| A report's own name | `report` | 10 | live-confirmed |
| A report parameter's label | `report_parmtr` | 31 | enum-confirmed |
| A control procedure's own name/description | `control_proc` | 46 | live-confirmed |
| A layout control / spacer / static label on a form | `gui_object` | 13 | live-confirmed; a literal single space can be an intentional blank, not untranslated — see below |
| A top-level menu's own name | `menu` | 294 | live-confirmed structurally (dedicated nav property); content not independently re-verified |
| A list-bar-style menu **group** header | `list_bar`, **not** `list_bar_grp` | 12 (not 78) | **live-confirmed trap, reconfirmed this session** (ids `CRM`/`finance`/`control`) |
| A list-bar-style menu **item** | no working path — shows the linked tab/report/task's own translation instead | 79 exists but is inert | live-confirmed, no consumer |
| A tile-style menu **group** header | `tile_grp` — its **own** type, no redirect | 87 | **live-confirmed this session** (ids `CRM`/`finance`/`employee`) — does *not* share the `list_bar_grp` trap |
| A tile-style menu **item** | no working path — same pattern as list-bar items | 84 exists but is inert | **live-confirmed this session** |
| A module/treeview group header | `module_grp` — its **own** type, no redirect | 6 | **live-confirmed this session** (id `control`) — does *not* share the `list_bar_grp` trap |
| A table-bound task's label (as shown on that table's screen) | no own type — shows the parent `task`'s (11) translation | 110 exists but has no translation of its own | **live-confirmed this session** — `tab_task` has no `transl_object` navigation property at all |
| A role's own name/description | `role` | 707 | enum-confirmed only; `role` entity not exposed by any domain reachable this session, so UI consumption is unverified — confirm live before trusting |
| A screen type's own name | `screen_type` | 164 | enum-confirmed |
| A tab variant's own name | `tab_variant` | 281 | enum-confirmed |
| A scheduler view's own name | `scheduler_view` | 971 | enum-confirmed |
| A conditional layout's own name | `conditional_layout` | 44 | enum-confirmed |
| A help index entry | `help_index` | 20 | enum-confirmed |
| A process flow's own name | `process_flow` | 29 | enum-confirmed |
| A cube view's own name | `cube_view` | 26 | **live-confirmed this session** |
| A cube field's own label (dimension or value) | `cube_field` | 25 | **live-confirmed this session; separate from the underlying column's own translation** — see trap below |

"Enum-confirmed" means the value is correct per a live fetch of the `type_of_object` enum itself,
but this session didn't independently verify which type the UI actually renders from — apply the
same bare-id, multi-type diagnostic used to catch the `list_bar_grp`/`tab_label` traps before
trusting a write to one of these rows blindly.

**A single column can own more than one `transl_object`, and they are not all the same
`type_of_object`.** `transl_object`'s navigation properties `detail_ref_transl_object_col`,
`detail_ref_transl_object_col_alt_transl_col_id`, `detail_ref_transl_object_col_form_next_grp_label`,
`detail_ref_transl_object_col_grid_next_grp_label`, and `detail_ref_transl_object_col_next_tab_label`
are five distinct back-references, confirming `col` carries five separate translatable-lookup fields
(its own label, `alt_transl_col_id`, `form_next_grp_label`, `grid_next_grp_label`, `next_tab_label`) —
but **the navigation property name does not tell you the target's `type_of_object`, and assuming they
all share `type_of_object = col` is wrong and was shipped as an error in an earlier version of this
skill.** Confirmed live: a column's own label and `alt_transl_col_id` are genuinely `type_of_object =
col` (1) — but `form_next_grp_label`/`next_tab_label` (form group headers and collapsible-tab-section
labels) resolve against a **separate, dedicated `type_of_object = tab_label` (33)**, keyed the same way
(`transl_object_id` = the literal label text, e.g. `general`, `status`, reused model-wide) but in its
own key space, entirely independent of any column that happens to share that same literal name.
**This is a real, live trap**: setting `next_tab_label = 'status'` on a column auto-generates a
`type_of_object = tab_label` row for `transl_object_id = 'status'` — a *different* row from the
`type_of_object = col` row for a column literally named `status`, even though both show up under the
identical-looking key `transl_object_id = 'status'`. Writing a translation to the wrong type silently
looks like it worked (commits cleanly, no error) while the actual UI-visible tab/group label stays on
its bracket placeholder. **Before translating a `form_next_grp_label`/`next_tab_label` value, verify
which `type_of_object` it actually resolves to live** (query `transl_object` filtered to the literal
label text across a couple of plausible `type_of_object` values, e.g. `col` and `tab_label`, and see
which one already carries a `[bracketed]` placeholder for that exact table/column) rather than
assuming either the generic `type_of_object` table above or this skill's own prior text. The same
per-field type_of_object caveat applies to `report_parmtr`/`task_parmtr`'s own `alt_transl_..._id`/
`form_next_grp_label`/`next_tab_label` fields and to tasks/reports' `alt_transl_confirm_button`/
`alt_transl_cancel_button` — don't assume any of them share the parent object's own `type_of_object`
without checking. Don't assume a parent object has exactly one translation either — check its own
navigation properties for `alt_transl_*`/`*_next_grp_label`/`*_next_tab_label`/`*_confirm_button`/
`*_cancel_button` fields before concluding a translation is missing.

**The same mismatch hits menu groups: a `list_bar_grp`'s real, UI-rendered translation is not
`type_of_object = list_bar_grp` — confirmed live.** Creating a group through its proper
group-creation task auto-generated the real, bracket-placeholder translation row at
`type_of_object = list_bar` (12), not `list_bar_grp` (78) as the enum name would suggest. Manually
adding a `transl_object` at `type_of_object = list_bar_grp` and translating that commits cleanly
with no error, but is inert — the actual group header stays on its placeholder at type 12. This was
caught by finding a **pre-existing** group in the same menu that had been mistranslated exactly this
way (presumably in an earlier session): its `list_bar_grp`-typed row held real text, while its
`list_bar`-typed row — the one actually shown to users — was still `[bracketed]`. **The general
diagnostic**: query `transl_object` filtered to just the bare id (no `type_of_object` filter) to see
every type_of_object it's registered under, then cross-check each row's `insert_date_time` against
when the owning object was actually created — the row created at the same moment as the object itself
is the real one; anything added later by hand is a candidate decoy. **Relatedly, `type_of_object =
list_bar_item` (79) accepts writes but has no corresponding navigation property on `transl_object` at
all** (unlike `list_bar_grp`, which does have one, even though it turned out to be the wrong type) —
no working consumption path was found for it. A menu item pointing at a table/report appears to
simply display that table/report's own translation; don't spend time translating `list_bar_item`
directly for that case.

**This `list_bar_grp` redirect is the exception, not a general "all menu-group types redirect"
rule — confirmed live.** Sibling group types were checked the same way this session and neither
shares the trap: `tile_grp` (87) group ids `CRM`/`finance`/`employee` each carry their
bracket-placeholder row at `type_of_object = 87`, their own type; `module_grp` (6) id `control`
carries its row at `type_of_object = 6`, also its own type. So `tile_grp`/`module_grp` group
headers translate directly under their own type — only `list_bar_grp` needs the `list_bar`
substitution above. `tile` (84), the tile equivalent of a `list_bar_item`, has the same "no
working path" shape as `list_bar_item`: no `transl_object` navigation property, and a real tile
(e.g. `customer_contact_person`) resolves only to its linked table's own `tab` (0) translation —
don't spend time translating `tile` directly either.

**A table-bound task has no translation of its own.** `tab_task` (`type_of_object = 110`) has no
`transl_object` navigation property at all — confirmed live via its entity definition. The label
shown for a task on a table's screen is the parent `task`'s own `type_of_object = 11` translation;
searching for a `tab_task`-scoped row is a dead end by design, not a missing-object case.

**A cube field's own label is a separate `transl_object` from its underlying column's — even when the
field's `col_id` points straight at that column.** Confirmed live: translating the column
(`type_of_object = col`, 1) does nothing for the label shown in the cube's dimension/value picker or
pivot headers — that's a fully independent row under `cube_field`'s own type (25), keyed by the
field's `cube_field_id`, not its `col_id`. The same applies in reverse: several `cube_field` rows can
share one `col_id` (a date column split into year/quarter/month interval fields, or a rollup/editable
pair pointing at the same measure column) and each needs its **own** `type_of_object = 25`
translation — translating one does not translate the others.

**Renaming an object via its dedicated rename task doesn't rename or remove its old translation.**
Confirmed live on a cube field renamed through its own rename task: the functional references to the
old id (e.g. the field's use elsewhere in the cube's views) were correctly cascaded to the new id, but
the old id's `transl_object`/`transl_object_transl` rows were left behind untouched — an orphan under
a name nothing points to anymore. After renaming any translatable object through its dedicated rename
task, check for and delete the old id's translation rows rather than assuming the rename task handled
them.
