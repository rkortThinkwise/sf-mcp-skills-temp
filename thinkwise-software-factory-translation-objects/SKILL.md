---
name: thinkwise-software-factory-translation-objects
description: Reference guide for translating model objects in a Thinkwise Software Factory model — the transl_object/transl_object_transl entities, appl_lang/branch_appl_lang, the approval_status review workflow, naming/plural/help-text conventions, and linking a base model to back-fill an officially supported language instead of hand-translating it. Use whenever an MCP connector with Software Factory access (e.g. sf_mcp, sf_dev_wiz_mcp, insights) reads, writes, generates, or reviews translations, or adds a new application language — before calling get_entity_definition/get_task_definition/execute_odata_query/stage_task against transl_object, transl_object_transl, appl_lang, branch_appl_lang, linked_model, or linked_model_available.
---

# Translating Objects in the Thinkwise Software Factory

Every translatable thing in a model — a table, a column, a domain element, a task, a report
parameter, a menu, a validation message, and roughly 150 other concepts — gets exactly one
`transl_object` row, keyed by `(model_id, branch_id, type_of_object, transl_object_id)`. Each
`transl_object` then has one `transl_object_transl` child row **per configured application
language**, keyed by the same four fields plus `appl_lang_id`, holding the actual translated text.
Everything in this skill was confirmed live against a real connected model (`INSIGHTS`, branch
`MAIN`) via `sf_mcp` — not guessed from documentation.

Apply this whenever an MCP connector with Software Factory access is used to read, write, generate,
or review translations. Follow the connector's standard discovery→act flow; never guess entity/task/
property names. `transl_object`, `transl_object_transl`, and `branch_appl_lang` live in a
`manage_translation`-style domain (confirmed `sf/manage_translation` live); the master `appl_lang`
list (the global catalogue of IETF language tags, independent of any model) lives in a
`manage_datamodel`-style domain instead — try these directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session. This skill
is connector-agnostic: it names entities, tasks, and fields, not any one connector's literal tool
names.

## Golden rule — several moves here need the user's word, not the assistant's judgment

The mechanics of reading and writing translation rows are simple API calls. A handful of moments in
that flow are design or process decisions that belong to the user, not something to resolve
silently by picking whatever seems safest — the `thinkwise_software_factory_mcp_base`
"Confirm-before-mutate" convention applied at this skill's own grain:

- **Hand-creating a `transl_object`.** The platform generates these automatically in almost every
  case; hand-inserting one is a last resort and requires confirming with the user first — the object,
  the `type_of_object` you intend to use, and why the automatic paths didn't apply (see "Before
  anything else" below).
- **Deciding `help_text` coverage scope.** Filling in `help_text` for one language is an implicit
  promise to cover every configured `branch_appl_lang` or knowingly leave the rest without it — ask
  the user which before writing any `help_text` (see "Help text" below).
- **Disapproving a translation.** `task_mark_transl_object_transl_disapproved` has a mandatory
  `feedback` string — there is no way to disapprove without a reason, so that reason has to come from
  the user (or genuinely reflect their stated concern), not be invented to satisfy the parameter.

Only skip asking when the user has already stated the answer unambiguously.

## Before anything else: find the existing object — don't create one

Nearly every translatable object already has an auto-generated `transl_object` row the moment
it's created in the model — even if nobody has ever translated it, that row exists, holding
`[bracketed]`-placeholder text (see "Detecting untranslated objects" below). **If you've been
asked to translate something, a row for it almost certainly already exists.** The job is almost
always to find that row and overwrite its placeholder `transl_object_transl` text — not to add a
new `transl_object`. The platform generates translation objects; you very rarely should.

Mandatory sequence, in order:

1. **Identify the object's kind and id** (a table, a column, a menu item, a role, …).
2. **Look up the candidate `type_of_object` value(s)** in the lookup table below. If that kind is
   flagged as having a trap (its real translation lives under a *different* `type_of_object` than
   its own name suggests), check the trap target **first**, not the literal name.
3. **Query `transl_object` filtered to just the bare `transl_object_id`** (no `type_of_object`
   filter) to see every type it's actually registered under, and cross-check the results against
   the lookup table — don't trust the first hit or assume "no row at the obvious type" means "no
   row anywhere."
4. **If a row exists, that's the one to edit.** Fetch its `transl_object_transl` for the target
   language(s). If the field(s) show the `[bracketed]` placeholder, overwrite that text — that's
   the actual translation work. If it already holds real, non-bracketed text, it's already
   translated; leave it alone unless explicitly asked to change existing text.
5. **Only if step 3 finds nothing at all**, across every plausible type for that kind, is the row
   genuinely missing. Reach for the platform's own generation path first — the object's own
   `generate_transl_object`-style creation parameter, or the matching `task_generate_transl_objects`
   variant (see "Generating translation objects" below). That's still the platform creating the
   row, not you hand-authoring one.
6. **Hand-inserting a `transl_object` row directly is the last resort**, and requires confirming
   with the user first: name the object, the `type_of_object` you intend to use, and why the
   automatic paths in step 5 didn't apply. Never create one silently — see "Backfilling a missing
   `transl_object` by hand" below for the mechanics once confirmed.

**About to add a new `transl_object`? Stop.** Re-run the bare-id search from step 3 across the
lookup table's candidates first. In the overwhelming majority of cases the row already exists —
either under a `type_of_object` you weren't expecting, or holding placeholder text that was
mistaken for "missing."

## Bulk-importing translations in one call — check for a custom task first

Some models have a rare, custom-built task that upserts a whole batch of translations in one JSON
call instead of writing each row individually. For the full workflow, JSON payload shape, and its
verified gotchas, read `references/bulk_import.md` before assuming this path is or isn't available
in the model you're working with.

## The two core entities

### `transl_object` — one row per translatable object

| Field | Type | Notes |
|---|---|---|
| `model_id`, `branch_id` | key | |
| `type_of_object` | key, `Edm.Int32` enum | Which kind of model concept this is — see below. |
| `transl_object_id` | key, string | The object's own ID (a column's `col_id`, a table's `tab_id`, a domain element's `elemnt_id`, …). **Not qualified by owner** — see the naming gotcha below. |
| `transl_object_description` | string | Free-text description of the translation object itself (not a translation). |
| `insert_user`/`insert_date_time`/`update_user`/`update_date_time` | trace | |
| `generated_by_control_proc_id` | string | Set when the object/its translations were produced by a control procedure rather than authored directly. |

Bound tasks: `task_generate_transl_objects` (zero params — backfills a missing `transl_object` for
an object that already exists in the model but has none yet, e.g. something created outside the
normal flow via a dynamic model), `task_show_history`, `task_unlink_generated_object`.

`transl_object` carries roughly ninety `detail_ref_transl_object_<parent>` navigation properties (to
`col`, `tab`, `dom_elemnt`/`elemnt`, `task`, `report`, `menu`, `cube_view`, `screen_type`,
`conditional_layout`, `role`, and many more) — this is how a `transl_object` row is actually reached
from its owning model object; don't try to derive `transl_object_id` from the parent's key structure
by convention, confirm it via the correct navigation property or by reading `transl_object_id`
directly off a query result.

### `transl_object_transl` — one row per object per language

| Field | Type | Shown where (per platform docs, cross-checked live) |
|---|---|---|
| `model_id`, `branch_id`, `type_of_object`, `transl_object_id` | key | Same as parent `transl_object`. |
| `appl_lang_id` | key, string | IETF tag, e.g. `en-US`, `nl-NL`. |
| `transl` | string | General-purpose single label; the fallback used wherever no more specific field is set. |
| `transl_form` | string | Form labels, formlists, task/report parameter pop-ups. |
| `transl_grid` | string | Grid column headers. |
| `transl_card_list` | string | Card list headers (undocumented in the platform docs pulled for this skill, but a real, distinct field — confirm intent with the user before assuming it always mirrors `transl`). For what `card_list_label`/`card_list_order_no`/etc. actually control on the card itself, see `thinkwise_software_factory_subject_components`. |
| `transl_plural` | string | List/detail tab headers, card list headers — see Plural forms below. |
| `tooltip_text` | string | Hover hint; basic HTML allowed. |
| `help_text` | string | Longer-form, feeds Help screens — see Help text below. |
| `approval_status` | `Edm.Byte` enum: `not_yet_approved`=0, `approved`=1, `disapproved`=2 | Per-language, per-object review state. |
| `feedback` | string | Reviewer's note when disapproving. |
| `insert_user`/`insert_date_time`/`update_user`/`update_date_time` | trace | |
| `generated_by_control_proc_id` | string | |

**Which text fields are actually live depends on `type_of_object`.** The API marks each of
`transl`/`transl_form`/`transl_grid`/`transl_card_list`/`transl_plural` as editable-and-mandatory or
hidden per record, and this varies by the owning object's `type_of_object` rather than being fixed —
confirmed live: a `tab` (0) row carries `transl`+`transl_plural`; a `col` (1) row carries
`transl`+`transl_form`+`transl_grid`+`transl_card_list` (no `transl_plural`); a `task_parmtr` (32) row
carries only `transl`+`transl_form`; most other types (`dom_elemnt`, `task`, `list_bar`, `menu`,
`scheduler_view`, …) carry `transl` alone. Fetch the record first and set only the fields it reports
as editable/mandatory — writing a fixed set of "all five fields" on every object either fails on
fields the object doesn't have, or silently sets fields the UI never shows.

Bound tasks (all zero-parameter except the one noted): `task_enrichment_auto_transl` ("translate with
AI" for this one row — drafts from an already-translated language), `task_generate_transl_objects`,
`task_mark_transl_object_transl_approved`, `task_mark_transl_object_transl_disapproved` (**mandatory
`feedback` string parameter** — cannot disapprove without a reason), `task_show_history`,
`task_transl_selected_objects` ("translate by ID" — derives a label from the object ID: underscores
to spaces, first letter capitalized), `task_unlink_generated_object`.

**Verified live** — table `activity` in model `INSIGHTS`, branch `MAIN` — surfaced three points made
elsewhere in this skill at once:

- **Eight configured languages**, not the shipped six — including a regional variant (`pt-BR`, not
  plain `pt`) and a non-Latin script (`ja-JP`).
- `transl_plural` authored independently per language, not derived from `transl` (see Plural forms
  below).
- `help_text` filled for only 2 of the 8 languages (`en-US`, `nl-NL`) — see Help text below.

## Detecting untranslated objects — look for the bracketed placeholder, not a blank field

A newly generated `transl_object_transl` row is not blank. The platform auto-fills every mandatory
text field with the object's own ID wrapped in square brackets — `[activity]`, `[employee_id]`,
`[employee_schedule_add_activity]` — as a fallback label, and that bracketed placeholder is what
still shows in the Software Factory's own UI for text nobody has actually translated. Confirmed live: querying
`transl_object_transl` for one language in a small custom model returned zero rows with `transl`
null or empty, but fifty-five rows across `transl`/`transl_form`/`transl_grid`/`transl_card_list`/
`transl_plural` matching `startswith(field,'[') and endswith(field,']')`. That bracket test is the
real "still needs translating" query — a null/empty check alone will under-report almost everything.

**Exception, also confirmed live:** a `gui_object` (`type_of_object = 13`) with `transl` set to a
single literal space (not empty, not bracketed) was not a missed translation — it was an
intentionally blank label on a layout/spacer control, where a real caption would introduce an
unwanted header in the UI. Before "fixing" any blank-looking (not bracketed) translation, check
whether the owning object is this kind of spacer/filler control rather than assuming it's simply
untranslated.

## `type_of_object` — one enum, ~150 values, spanning almost the whole model

`type_of_object` is not specific to tables/columns/domain elements — it is the same enum used to key
*every* translatable concept in the platform, and it grows across platform versions. **Always fetch
the current values live** — `get_entity_definition` (or the connector's equivalent metadata call)
against `transl_object`/`transl_object_transl`'s `type_of_object` property/domain — rather than
trusting any hardcoded list, including the one in the reference file below.

For the full `type_of_object` value table and its known traps (`tab_label`, `list_bar_grp`,
`cube_field`, rename-orphan, and more), read `references/type_of_object_lookup.md` before writing a
`type_of_object` filter or interpreting a lookup result.

## Naming — the object ID *is* the translation key

Because `task_transl_selected_objects` ("translate by ID") derives a label straight from
`transl_object_id`, a well-named object is half-translated before anyone types anything —
`sales_order_line_status` becomes "Sales order line status" for free.

**Gotcha, confirmed live: `transl_object_id` for a column is the bare column ID — not qualified by
table.** Querying the real `INSIGHTS` model for `col` rows with `col_id eq 'description'` returns ten
different tables (`activity`, `country`, `declaration`, `declaration_lines`, `employee_function`,
`employee_item`, `hour`, `hours_with_customer`, `meeting`, `project`, …) all sharing that column name.
Querying `transl_object_transl` for `type_of_object eq 1 and transl_object_id eq 'description'`
returns exactly **one** row per language — meaning all ten tables' `description` columns render the
same translated text, and editing any one of them changes it everywhere at once. This is by design
when the columns really do mean the same thing (a generic "description" field genuinely should read
identically everywhere) — but it's a live trap the moment two same-named columns need to diverge.

Two fixes, confirmed against the platform's own translation model:

- **Rename the column** so its ID is specific to what it actually holds (`loon_component_code`
  instead of a bare `code` reused elsewhere with different meaning) — the default choice, and it also
  sits well with `task_transl_selected_objects` auto-deriving a better label for free.
- **Point `col.alt_transl_col_id` at a different column's ID** to deliberately *share* a translation
  with something else, or add a genuinely new column and translate it independently, if the source
  column must keep its generic ID for another reason (e.g. it's shared through a domain on purpose).

**Domain elements** (`type_of_object = dom_elemnt`, value 2): the `transl_object_id` is the element's
own ID. Real IDs pulled live from `INSIGHTS` include both plain descriptive IDs (`approved`,
`approve_working_hours`) and numeric-prefixed ones (`1_project`, `2_customer`) — the latter pattern
sacrifices some of the "ID carries all descriptive meaning, database value stays a meaningless
sequential integer" convention (see the `thinkwise_datamodeling_guidelines` skill's Domain elements
section) for explicit, stable ordering. Match whatever convention the model already uses rather than
mixing both within one domain.

## Avoid the literal word "ID" in translated text

Translated labels (`transl`/`transl_form`/`transl_grid`/`transl_card_list`/…) should read as natural
language for an end user, not restate the database naming convention. An object like `employee_id`
should normally translate to "Employee", not "Employee ID" — the underlying column name already
encodes that it's an identifier; the label doesn't need to repeat it. Only keep "ID" in the
translated text when it's genuinely the only unambiguous option — e.g. an external reference number
where "ID" (or a more specific term like "order number") is the actual meaning being conveyed, or a
screen that shows a name and a numeric identifier side by side and needs the second one distinguished
from the first.

## Plural forms — authored per language, not derived

`transl_plural` is its own field, filled independently per `appl_lang_id` — the live `activity`
example above shows eight different, grammatically-correct plural strings (`Aktivitäten`,
`Activities`, `Actividades`, `Activités`, `Attività`, `アクティビティ`, `Activiteiten`, `Atividades`),
none of which the platform derived from `transl`. Where it's used: list/detail tab headers, card list
headers — anywhere the UI names a collection of rows rather than one row.

**Link tables shown as a detail tab get a directional plural.** Per platform docs: write
`transl_plural` as `singular/plural` (e.g. `Persons/Companies` for a person↔company link table) — the
UI shows the second half when the tab is opened from the "singular" side (a person's record shows a
"Companies" tab) and the first half from the other side (a company's record shows a "Persons" tab).

Because pluralization grammar differs per language (English's irregular plurals, German/Dutch
compounding, gendered agreement in French/Spanish/Portuguese/Italian), a plural correct in one
language says nothing about another. If `task_enrichment_auto_transl` (AI) is used to backfill a
plural into a new language, treat it as a draft, not a commit — review it before approving, same as
any other AI-drafted translation.

**Verify after a combined write, confirmed live:** another instance of the general "a multi-field
write can silently drop one field" behavior in `thinkwise_datamodeling_guidelines`'s quirks section —
setting `transl` and `transl_plural` together in one write occasionally left `transl_plural` reverted
back to equal the just-set singular `transl` instead of the plural value that was also specified in
the same write — reproducible on 2 of 6 table records in one session, not a one-off typo. Re-read the
field back after this kind of combined write and re-apply `transl_plural` if it doesn't match before
committing; don't assume a multi-field write that returned success actually applied every field as
given. **Re-tested and fixed, verified live**: setting `transl_plural` *after* `transl` in the same
combined call avoided the revert entirely — both fields held correctly with no follow-up patch needed.
Order the properties this way instead of budgeting for a second corrective write.

## Help text — leave `approval_status` review aside, default to empty

`help_text` is a distinct, longer-form field tied to Help screens — not a longer tooltip. The live
`activity` example makes the real cost concrete: `help_text` is filled with a genuine, well-written
definition-plus-example for exactly **two of the eight configured languages** (`en-US`, `nl-NL`); the
other six (`de-DE`, `es-ES`, `fr-FR`, `it-IT`, `ja-JP`, `pt-BR`) simply have none. That's not
necessarily a bug — it may be a deliberate call that only two languages' user bases need the extra
explanation — but it is exactly the failure mode to design against: **every `help_text` filled in one
language is an implicit promise to either fill it in every other configured language, or accept that
most of your users never see it.**

**Convention: leave `help_text` empty by default.** Reach for it only when a field's behavior is
genuinely non-obvious — a calculation the label can't convey, a workaround, a non-obvious
precondition. A clear object name (see Naming above) plus a short `tooltip_text` covers the large
majority of fields. Before filling in `help_text` for a new object, either commit to authoring it for
every language the branch has configured (query `branch_appl_lang` to know how many that is — see
below) or flag to the user that it will be English/source-language-only for now.

## Multiple languages

**Adding a new language? Check whether it's officially supported before translating anything by
hand.** Thinkwise ships a pre-translated base model per officially supported GUI language
(`GUI_TRANSL_<lang>`, plus a narrower `<RDBMS>_MSG_TRANSL_<lang>` for database error text on some
RDBMS/language combos) — linking and merging that base model back-fills the platform's own standard
translations for free. Manual translation via this skill is then only needed for the application's
**own** objects. See `references/officially_supported_languages.md` for the live-verified language
↔ base-model mapping and the full link → merge → generate recipe before doing any manual work on a
newly-added `branch_appl_lang`.

**`appl_lang`** (in a `manage_datamodel`-style domain, model-independent) is the global catalogue:
`appl_lang_id` (the IETF tag, e.g. `en-US`, key) and `appl_lang_description`. **`branch_appl_lang`**
(in `manage_translation`) links one of those to a specific `(model_id, branch_id)` — it's how a branch
declares which languages it actually supports; keyed by `(model_id, branch_id, appl_lang_id)`, no
translatable fields of its own beyond `generated_by_control_proc_id`. Query `branch_appl_lang` first
to know how many languages a `help_text` decision (above) or a plural-review pass (above) actually
needs to cover — don't assume the shipped six (`nl-NL`/`en-US`/`de-DE`/`fr-FR`/`es-ES`/`pt-PT`); the
live `INSIGHTS` example has eight, including `it-IT`, `ja-JP`, and `pt-BR` instead of `pt-PT`.

**Cleaning up orphaned translation objects.** `branch_appl_lang` carries a zero-parameter bound task,
`task_delete_unused_transl_objects` ("Delete unused translation objects — Delete unused translation
objects for the entire branch"), verified live: it deletes every `transl_object`/`transl_object_transl`
row whose underlying model object no longer exists, branch-wide, in one call. The most common source of
these orphans is renaming something — a rename task changes an object's id but doesn't clean up the
*old* id's translation rows, which then sit around indefinitely holding stale bracket-placeholder text
under an id nothing points to anymore. Reach for this task after any rename, or as a periodic branch-wide
sweep, rather than hunting down a specific orphaned row by hand once you happen to notice one. **This
task can be entirely invisible without the right Software Factory role/rights on the connected
account** — confirmed live: `branch_appl_lang`'s bound-task list came back empty and a direct lookup by
this task's exact name returned a hard "not found," indistinguishable from the task simply not
existing, until the account's rights were adjusted — see `thinkwise_datamodeling_guidelines`'s
role/rights quirk for the general version of this trap.

**Generating translation objects.** New objects normally get a `transl_object` automatically on
creation — confirmed live on `tab`'s own `task_create_tab_variant`, which carries a
`generate_transl_object: Edm.Boolean` parameter controlling exactly this. For anything created without
going through such a task (most commonly dynamic-model objects), call the relevant
`task_generate_transl_objects` variant afterward — it exists both as a generic bound task on
`transl_object`/`transl_object_transl` and as dozens of narrower unbound variants scoped to a specific
parent (`generate_transl_objects_help_index_help_index_id`,
`generate_transl_objects_tab_check_constraint_check_constraint_id`,
`generate_transl_objects_module_grp_module_grp_id`, and many more) — use
`search_domain_capabilities` in `sf/manage_translation` to find the variant matching the parent object
type actually in play rather than assuming the generic one always applies.

**Confirm the row is actually missing before reaching for this recipe.** Whether a dependent-record-added
`list_bar_grp`/`list_bar_item` gets an auto-generated `transl_object` for free is inconsistent across
sessions — one session found zero rows immediately after creating and committing the group/item (the
case this recipe was built for), another found a real bracket-placeholder already there with no extra
steps, for the same kind of object created the same way. Run the bare-id query from step 3 at the top
of this skill first; only fall through to hand-backfilling below if that query genuinely comes back
empty.

**Backfilling a missing `transl_object` by hand — the working recipe, verified live for a
`list_bar_item` (a case earlier assumed to be a dead end):**

1. **Add the `transl_object` row directly** (`model_id`, `branch_id`, `type_of_object`,
   `transl_object_id` matching the target object's own id) — this succeeds as a plain add, no bound
   task needed.
2. **Run the *generic* `task_generate_transl_objects`, bound to that exact `transl_object` row you
   just created** (zero params). This is the key step that's easy to skip: it's not the narrow
   per-parent-type unbound variant (`generate_transl_objects_<parent>_<parent>_id`-shaped) — that
   narrow variant can fail on its own hidden mandatory `branch_id` with no exposed way to target one
   record. The generic bound task on `transl_object` succeeded where the narrow variant didn't, and
   produced the bracket-placeholder `transl_object_transl` row for every configured language.
3. **Edit that `transl_object_transl` row normally** (it's now an existing row addressed by its full
   key including `appl_lang_id`, not a fresh add) — the fields patch and commit like any other
   translation.

**Why hand-adding `transl_object_transl` directly (skipping step 2) fails**: on a brand-new add,
`appl_lang_id` shows as hidden-yet-mandatory and rejects any attempt to patch it — a real, confirmed
dead end for that specific path. The fix isn't to abandon the API and go to the Software Factory's own
UI, it's to route through step 2's generate task instead of trying to hand-craft the child row.

If a future case still has no working path after trying all three steps above, *then* treat manual
translation directly in the Software Factory as the practical fallback — but confirm that by actually
attempting this recipe first, since it resolved what previously looked unrecoverable.

**Review workflow.** `approval_status` starts `not_yet_approved` on a new/changed translation.
`task_mark_transl_object_transl_approved` (zero params) approves it; disapproving requires
`task_mark_transl_object_transl_disapproved` with a mandatory `feedback` string — there is no way to
disapprove without leaving a reason. Treat `approval_status` as the gate for whether a set of
translations is safe to consider final, especially before a merge — an edited, previously-approved
translation resets to `not_yet_approved` (per platform docs; not independently re-verified live in
this session), so a stale "approved" count is a real risk signal, not just paperwork.

**Fallback language.** Referenced in platform documentation (used when a user's own language has no
translations in a given application) but its storage location was **not** found during live
verification in either `sf/manage_translation` or `sf/manage_datamodel` — it did not surface as a
field on `branch`, `appl_lang`, or `branch_appl_lang`, and no `sf_configuration`-style entity was
reachable from either domain in this session. Confirm its actual entity/field (or that it's a setting
only exposed in the Software Factory's own UI, with no exposed API surface) live before scripting
anything that depends on it, rather than assuming a name from documentation alone.

**Parameterized labels.** Per platform docs, `transl_form` (and the group/section label fields
surfaced via `col`/`report_parmtr`/`task_parmtr`'s `form_next_grp_label`/`grid_next_grp_label`
lookups above — see `thinkwise_software_factory_subject_components` for what those group/section
fields actually do on a Form or Grid) can carry `{placeholder}` tokens resolved with locale-aware
date/number formatting.
Not recursive, resolved exactly once, and an undefined/null parameter is silently dropped rather than
breaking the string — not independently re-verified live in this session, treat as documented
behavior to confirm if a parameterized label misbehaves.

## Pre-flight checklist

- **Adding a new language that's on the officially supported list?** Link its `GUI_TRANSL_<lang>`
  base model (and matching `<RDBMS>_MSG_TRANSL_<lang>` if one exists for the branch's RDBMS), merge
  it into the work model, and generate — before doing any manual translation. See
  `references/officially_supported_languages.md`.
- **Never create a `transl_object` as your first move.** Search first: query the bare
  `transl_object_id` across every plausible `type_of_object` from the lookup table above, not just
  the obvious one, and look for an existing (possibly `[bracketed]`) row before concluding one is
  missing. Hand-creating a `transl_object` is a last resort that requires confirming with the user
  first — see "Before anything else: find the existing object" above.
- **Confirm the domain keys before querying**: `transl_object`/`transl_object_transl`/
  `branch_appl_lang` in a `manage_translation`-style domain, `appl_lang` in a `manage_datamodel`-style
  domain — don't assume `sf/manage_translation`/`sf/manage_datamodel` hold on every connector.
- **`transl_object_id` for a column is the bare column ID, not qualified by its table** — before
  editing a column's translation, check whether other tables reuse the same `col_id` (query `col`
  filtered on that `col_id` across the model) and confirm the edit is meant to apply everywhere it's
  used, or rename/point `alt_transl_col_id` elsewhere first.
- **Check for `alt_transl_*`/`*_next_grp_label`/`*_next_tab_label`/`*_confirm_button`/
  `*_cancel_button` on the parent object** before concluding an object has only one translation — many
  do not.
- **`form_next_grp_label`/`next_tab_label` resolve against `type_of_object = tab_label` (33), not
  whatever type the owning object is** — confirmed live. Don't translate the owning column/parameter's
  own `type_of_object` row and assume that's the group/tab-section label; verify which `type_of_object`
  actually holds a `[bracketed]` placeholder for that literal label text before writing to it. Committing
  to the wrong type_of_object succeeds with no error and leaves the real UI-visible label untranslated.
- **A `list_bar_grp`'s real translation is `type_of_object = list_bar` (12), not `list_bar_grp` (78)**
  — confirmed live, the same shape of mismatch as `tab_label` above. Query all `type_of_object` values
  for the bare group id and cross-check `insert_date_time` against the group's own creation to find the
  real, auto-generated row before translating. `list_bar_item` (79) has no navigation property on
  `transl_object` at all and no known working consumer — a table/report-linked menu item just shows
  the linked object's own translation.
- **`tile_grp`/`module_grp`/`tab_task` do *not* share the `list_bar_grp` trap** — confirmed live.
  `tile_grp` (87) and `module_grp` (6) group headers translate under their own type directly, no
  redirect. `tile` (84) has the same "no working path" shape as `list_bar_item`. `tab_task` (110)
  has no `transl_object` navigation property at all — a table-bound task's label is the parent
  `task`'s (11) own translation, not a `tab_task`-scoped row.
- **A `cube_field`'s label (25) is independent of its `col_id` column's own translation (1)** —
  translating one does nothing for the other, and multiple `cube_field` rows sharing one `col_id`
  each need their own translation. `cube_view` is `type_of_object = 26`.
- **A rename task cascades functional references but not translations** — after renaming a
  translatable object through its dedicated rename task, check for and delete the old id's orphaned
  `transl_object`/`transl_object_transl` rows rather than assuming the rename handled them.
- **`transl_plural` is never derived — author it explicitly per language**, and for a link table shown
  as a detail tab, use the `singular/plural` directional format.
- **Avoid the literal word "ID" in translated text** (e.g. `employee_id` → "Employee", not "Employee
  ID") — only keep it when it's the sole unambiguous option.
- **To find objects that still need translating, look for `[object_id]`-bracketed placeholder text**
  (see "Detecting untranslated objects") — not blank/null fields — and don't reflexively overwrite a
  literal single-space value, which can be an intentional blank spacer label.
- **Field editability (`transl`/`transl_form`/`transl_grid`/`transl_card_list`/`transl_plural`)
  depends on `type_of_object`** — fetch the record and set only what it reports as editable, don't
  assume all five apply.
- **Set `transl` first and `transl_plural` last in the same combined write** — verified live, this
  avoids the revert that a separate follow-up patch was previously needed to catch.
- **Default `help_text` to empty.** Filling it in commits to either covering every language on the
  branch (`branch_appl_lang`) or knowingly leaving the rest without it — **ask the user** whether to
  commit to authoring `help_text` for every configured `branch_appl_lang`, or to explicitly accept
  English/source-language-only coverage, before writing any `help_text`, rather than picking one
  silently.
- **`task_mark_transl_object_transl_disapproved` requires a `feedback` string** — it will not commit
  without one.
- New object with no translations showing up? Run the matching `task_generate_transl_objects` variant
  for its parent type (via `search_domain_capabilities`) before assuming something else is broken —
  most object-creation tasks already do this via a `generate_transl_object`-style parameter, so check
  that first.
- **No `transl_object` exists at all yet?** Add it by hand, then run the *generic* zero-param
  `task_generate_transl_objects` bound to that row (not a narrow per-parent-type variant) — see
  "Backfilling a missing `transl_object` by hand" above. Don't jump to hand-adding
  `transl_object_transl` directly; that path's `appl_lang_id` field is hidden-yet-mandatory on add and
  will block the commit.
- **Fallback-language configuration's storage location is unconfirmed** — don't script against a
  guessed field/entity name for it; verify live first (or confirm with the user that it's only
  exposed in the Software Factory's own UI).
