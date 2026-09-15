# Adding an officially supported language — link a base model, don't hand-translate

Before manually translating a newly-added `branch_appl_lang` from scratch, check whether it's on
the **officially supported list**. Thinkwise ships every officially supported GUI language as its
own standalone, pre-translated **base model** — linking (and merging) that base model into the
work model backfills the platform's own standard translations for free. Manual translation, via
the rest of this skill, is then only needed for the application's **own** objects (its tables,
columns, custom tasks, custom messages) — never for anything the platform itself ships.

Confirmed live end-to-end against `meta_dev`/`sf/software_development`, including a full
link → generate → verify → unlink → regenerate → verify cycle on a real work model — not assumed
from documentation, and not just inferred from metadata.

## The base-model catalogue

`model` carries three flags: `base_model`, `default_base_model`, `thinkwise_base_model`. Querying
`thinkwise_base_model eq true` surfaces the platform's own shipped base models, including one
**GUI translation base model per officially supported language**:

```
model?$filter=thinkwise_base_model eq true and startswith(model_id,'GUI_TRANSL_')
```

**Live language ↔ base-model mapping** (confirmed via `i_model_appl_lang`, see below — don't derive
this from the `model_id` suffix, it's a hint, not a contract: `GUI_TRANSL_ENG`, not
`GUI_TRANSL_EN_US`):

| `appl_lang_id` | Base model |
|---|---|
| `en-US` | `GUI_TRANSL_ENG` |
| `nl-NL` | `GUI_TRANSL_NL` |
| `de-DE` | `GUI_TRANSL_DE` |
| `es-ES` | `GUI_TRANSL_ES_ES` |
| `fr-FR` | `GUI_TRANSL_FR_FR` |
| `hu-HU` | `GUI_TRANSL_HU_HU` |
| `it-IT` | `GUI_TRANSL_IT_IT` |
| `ja-JP` | `GUI_TRANSL_JA_JP` |
| `lt-LT` | `GUI_TRANSL_LT_LT` |
| `lv-LV` | `GUI_TRANSL_LV_LV` |
| `pt-BR` | `GUI_TRANSL_PT_BR` |
| `ro-RO` | `GUI_TRANSL_RO_RO` |
| `ru-RU` | `GUI_TRANSL_RU_RU` |
| `sk-SK` | `GUI_TRANSL_SK_SK` |
| `tr-TR` | `GUI_TRANSL_TR_TR` |
| `zh-CN` | `GUI_TRANSL_ZH_CN` |
| `cs-CZ` | `GUI_TRANSL_CS_CZ` |

**Re-verify this table live before trusting it** — the platform adds languages over time (this list
was current 2026-08), and `i_model_appl_lang` (a read-only "Supported language (Model interface)"
entity, keyed `model_id`/`branch_id`/`appl_lang_id`) is the ground truth per candidate base model:

```
i_model_appl_lang?$filter=model_id eq 'GUI_TRANSL_NL'
```

**A same-prefix trap, confirmed live**: `i_model_appl_lang` also returned a hit for
`GUI_TRANSL_HOTFIX` (→ `en-US`) — a model that does **not** appear in the `thinkwise_base_model`
list above. A `GUI_TRANSL_*` name prefix alone is not sufficient to identify an official language
base model; always cross-check against `thinkwise_base_model eq true` on `model` as well.

**Database error-message translations are a separate, narrower base model, per RDBMS, and only for
three languages.** `<RDBMS>_MSG_TRANSL_<LANG>` base models (`SQLSERVER_MSG_TRANSL_*`,
`ORACLE_MSG_TRANSL_*`, `DB2_MSG_TRANSL_*`) exist only for `ENG`/`DE`/`NL`. PostgreSQL has only
`POSTGRESQL_MSG_TRANSL_ENG` — **no German or Dutch Postgres variant exists**. Check which RDBMS the
target branch actually uses (its own linked `SQLSERVER_DB`/`POSTGRESQL_DB`/`ORACLE_DB`/`DB2_DB` base
model) and whether a matching `_MSG_TRANSL_<LANG>` model exists at all before promising database
error messages will be translated — for Postgres apps adding German/Dutch, they won't be from this
mechanism, full stop.

## The linking mechanism

`linked_model` (junction table: `work_model_id`, `work_branch_id`, `base_model_id`,
`base_branch_id`, `gen_order_no`; directly writable) is how any base model — translation or
otherwise — gets attached to a work model/branch. `linked_model_available` is a candidate-list view
over the same keys with a `selected` boolean toggle, scoped to one specific work model/branch.

**A typical linked-base-model set, confirmed live on two independent real work models**: alongside
the RDBMS base model (`SQLSERVER_DB`/`POSTGRESQL_DB`/etc.), a handful of standard platform base
models (`DEFAULT_THEMES`, `ENRICHMENTS`, `SCREEN_TYPES`, `VALIDATIONS`, an icon-set model, …) each
sit at their own `gen_order_no`, and each configured language gets its own `GUI_TRANSL_<lang>` entry
(plus a `<RDBMS>_MSG_TRANSL_<lang>` entry where one exists) slotted in alongside them — e.g. one
observed model with only `en-US` had just `GUI_TRANSL_ENG` + the matching `_MSG_TRANSL_ENG` linked;
another with `en-US` + a second language had **both** languages' `GUI_TRANSL_<lang>` **and**
`_MSG_TRANSL_<lang>` entries linked, each language's pair sitting at adjacent `gen_order_no` values.
This is the target shape: for every supported language, link its `GUI_TRANSL_<lang>` (and, if one
exists for the branch's RDBMS, its `<RDBMS>_MSG_TRANSL_<lang>`) with a `gen_order_no` slotted
alongside the model's existing base models.

**Live gotcha — the eligible `base_branch_id` is not always `MAIN`, and is not a constant per base
model.** Confirmed live: querying `linked_model_available` for one real work model, filtered to a
specific `GUI_TRANSL_<lang>` base model, returned exactly one candidate at a **non-`MAIN`** branch —
even though that base model's own `MAIN` branch was `active`/not archived — while a *different* real
work model already had a working link to that exact same base model at `base_branch_id = 'MAIN'`.
The eligible branch is evaluated per work model, not fixed per base model. Always query
`linked_model_available` scoped to the actual work model/branch to get today's real eligible
`(base_model_id, base_branch_id)` pair — never hardcode `MAIN`.

**Direct mutation of a `MAIN` branch needs the right connector permissions.** A `branch_appl_lang`
add against a `protected = false` `MAIN` branch initially failed with `branch_is_read_only`, and
`task_create_branch` returned `403 Forbidden` — both turned out to be a connector rights gap, not a
structural restriction: once the account was granted branch-mutation rights, the identical
`branch_appl_lang` add, the `linked_model` adds/deletes, and `task_delete_branch_appl_lang` on that
same `MAIN` branch all succeeded without incident. A separate `protected = true` `MAIN` branch was
not re-tested after the rights grant — whether a `protected = true` branch stays blocked regardless
of rights, or was hitting the exact same gap, is still open; confirm live on a `protected` branch
before assuming either way.

## The recipe

**Verified end-to-end live** on a real SQL Server work model (adding German, `de-DE`): every step
below actually ran and produced the expected result — this is not inferred from metadata.

1. **Confirm the language is officially supported and find its base model** — query `model` for
   `thinkwise_base_model eq true and startswith(model_id,'GUI_TRANSL_')`, then `i_model_appl_lang`
   for the exact `appl_lang_id` match. Also check whether a `<RDBMS>_MSG_TRANSL_<LANG>` base model
   exists for the branch's own RDBMS.
2. **Add the language**: add a `branch_appl_lang` row for the target `(model_id, branch_id,
   appl_lang_id)` — this entity is a plain writable weak entity under `branch` (see this skill's
   "Multiple languages" section for the entity shape); stage it under its parent via
   `parent_entity_set: branch`, `parent_key: {model_id, branch_id}`.
3. **Find the real link target**: query `linked_model_available` filtered to
   `work_model_id eq '<model>' and work_branch_id eq '<branch>' and base_model_id eq '<GUI_TRANSL_..>'`
   to get today's actual eligible `base_branch_id` — don't assume `MAIN`. Repeat for the
   `<RDBMS>_MSG_TRANSL_<LANG>` base model if one applies.
4. **Link it**: add a `linked_model` row with that exact `(base_model_id, base_branch_id)` pair and
   a `gen_order_no` consistent with the model's existing base models (see "a typical linked-base-model
   set" above — slot new translation base models near the existing ones, e.g. right after the
   already-linked `GUI_TRANSL_<other-lang>` entry). **On add, `base_model_id`/`base_branch_id` must
   be patched with `value_kind: "data"`** — the lookup's display column (`model_id_display`) doesn't
   match the raw model id, so a plain value is read as a display value and fails `lookup_not_found`.
5. **Generate the definition** (`add_job_to_generate_definition`, per
   `thinkwise-software-factory-deployment`). **No separate merge step is needed — generation does it
   for you.** Confirmed live: `definition_generation_step` for a real generation job includes a step
   literally named `merge_base_models_into_work_model` / "Copy base models" (`order_no = 40`,
   ordered early, right after `prepare`/`delete_generated_objects`/`delete_prog_objects`), and it
   completes (`status = successful`) as part of the ordinary generation run — `add_job_to_generate_definition`
   alone is sufficient. The standalone `task_merge_base_models_into_work_model` bound task still
   exists (e.g. to preview/apply a merge without running a full generation), but is not a required
   prerequisite step.
6. **Verify**: query `transl_object_transl` for the new `appl_lang_id`. Confirmed live: after linking
   `GUI_TRANSL_DE` + `SQLSERVER_MSG_TRANSL_DE` to a real work model and generating, real German text
   appeared for base-model-owned objects with no further action — e.g. `gui_object` (13) rows
   `abandon` → `Aufgeben`, `about` → `Über`, `action_not_allowed` → `Diese Aktion ist nicht zulässig`
   — while the work model's own custom tables (`tab`, type 0) stayed on their `[bracketed]`
   placeholders, exactly matching the scope boundary below.

## Undoing it — confirmed to actually revert, not just stop growing

Confirmed live: deleting the `linked_model` row(s) (`delete_record`) and running
`add_job_to_generate_definition` again **does** revert the merged-in base-model text — it's a real,
working undo, not merely "no further changes." Re-checking the exact same rows after unlink+regenerate
showed `abandon`/`about`/`action_not_allowed` all reset back to `[bracketed]` placeholders, all
timestamped within the second generation job's own `merge_base_models_into_work_model` step. The
underlying `transl_object_transl` **row itself is not deleted** (the object still needs a row for
every configured `branch_appl_lang`) — only its base-model-sourced text is cleared back to the
platform's own auto-placeholder. If the language itself should also go away (not just its borrowed
text), separately delete the `branch_appl_lang` row via the bound `task_delete_branch_appl_lang`
(confirmed live to remove it cleanly) and generate once more.

## Scope boundary — what this does not cover

Linking and merging a `GUI_TRANSL_<lang>` (and, where one exists, `<RDBMS>_MSG_TRANSL_<lang>`) base
model only back-fills translations for objects the base model itself owns: standard messages,
validations, screen types, and GUI framework labels shipped by the platform. It does **not**
translate the application's own tables, columns, custom tasks, custom messages, or any other object
that only exists in the work model — those still need the normal manual `transl_object_transl`
workflow described in the rest of this skill. Adding a language is never "fully done" by linking
the base model alone.
