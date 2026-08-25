# Bulk-importing translations in one call — check for a custom task first

Some models have a **custom-built** task (not a standard platform feature — added on a per-model
basis, so it may or may not exist in the model you're working with) that accepts a single JSON
payload describing a batch of `transl_object`/`transl_object_transl` rows and upserts all of it in
one call, instead of writing each translation individually. If present, it's typically named
something like `bulk_import_translation_objects` and bound to the `branch` entity.

**Check availability before assuming it exists** — via the connector's task-discovery flow — and
**confirm its exact task/parameter names and JSON shape via `get_task_definition`/its own
documentation before calling it**; it's hand-written and can differ from or extend what's shown here.

As built in one reference model: parameters `model_id`, `branch_id`, `json_payload` (a large text
parameter), with this shape:
```json
{
  "objects": [
    {
      "type_of_object": 0,
      "transl_object_id": "...",
      "transl_object_description": "...",
      "translations": [
        { "appl_lang_id": "en-US", "transl": "...", "transl_plural": "...", "approval_status": 0 }
      ]
    }
  ]
}
```
`type_of_object` must be the **current live integer value** for the concept (see `type_of_object`
below) — never hardcode it from documentation or a previous session. Every translation text field
(`transl`/`transl_form`/`transl_grid`/`transl_card_list`/`transl_plural`/`tooltip_text`/`help_text`)
is optional per row; `approval_status` defaults to `0` (not yet approved) when omitted. Upsert-only,
matched on `(type_of_object, transl_object_id[, appl_lang_id])` — never deletes. The same
`transl_object_id`-sharing behavior applies as anywhere else (see Naming below): the task doesn't
special-case it, it just upserts whatever key the JSON gives it.

**"Optional per row" means the task won't error if you omit a field — it does not mean the object
ends up fully translated if you do.** Verified live: supplying only `transl` for a batch of `col`-type
(`type_of_object = 1`) objects committed cleanly, but left `transl_form`/`transl_grid`/
`transl_card_list` `null` for every one of them, because this kind of custom bulk-upsert task writes
exactly the fields it's given and has no equivalent of the platform's own bracket-placeholder
auto-fill on creation. Before calling it, check which fields the target `type_of_object` actually
needs (see "Which text fields are actually live depends on `type_of_object`" further below) and include all of them in the
payload — don't assume `transl` alone is enough just because the task accepts it without complaint.

**The JSON shape above is a minimal illustrative example, not the ceiling.** Confirmed live in a
second, independently-built instance of this task: it accepted every field in the full list above
(`transl_form`/`transl_grid`/`transl_card_list`/`tooltip_text`/`help_text`/`approval_status`), not
just `transl`/`transl_plural` — don't assume a task instance is limited to whatever subset a past
example happened to show. Read the actual task's own template/definition to confirm which fields a
given instance accepts before assuming the minimal example is exhaustive.

**If no such task is found, or calling it fails, fall back to the normal flow**: create/update each
`transl_object` and `transl_object_transl` row individually following the rest of this skill.
