# Bulk-importing a whole data model in one call — check for a custom task first

Some models have a **custom-built** task (not a standard Software Factory feature — added on a
per-model basis, so it may or may not exist in the model you're working with) that accepts a single
JSON payload describing a batch of domains/tables/columns/indexes/references and upserts all of it
in one call, instead of creating each object individually via the flow described in the rest of this
skill. If present, it's typically named something like `bulk_import_data_model_json` and bound to
the `branch` entity.

**Check availability before assuming it exists** — via the connector's task-discovery flow
(`search_capabilities`/`search_domain_capabilities`/`get_task_definition`, or equivalent) — don't
assume it's there just because a previous session built one, in this model or a different one; it
is not a platform feature. **If found, confirm its exact task/parameter names and JSON shape via
`get_task_definition`/its own documentation before calling it** — it's hand-written and can differ
from or extend what's shown here.

**This kind of task may live in its own dedicated domain, separate from the data-modeling domain
routine capability searches default to.** Verified live: a capability search scoped to the
data-modeling domain found nothing, while a full domain listing filtered by keyword turned up the
task immediately in a completely separate, otherwise-unrelated-sounding domain. If a targeted search
for a known task name comes up empty, fall back to listing all domains and grepping their
descriptions before concluding the task doesn't exist.

As built in one reference model: parameters `model_id`, `branch_id` (target scope) and
`json_payload` (a large text parameter), with this shape:
```json
{
  "domains": [ { "dom_id": "...", "dttp_id": "...", "length": 0, "prec": 0, "mand": true, "dom_description": "..." } ],
  "tables": [
    {
      "tab_id": "...",
      "type_of_table": 0,
      "tab_description": "...",
      "columns": [ { "col_id": "...", "dom_id": "...", "primary_key": true, "mand": true, "identity_col": false, "col_description": "..." } ],
      "indexes": [ { "indx_id": "...", "unique_indx": true, "columns": ["col_id", "..."] } ]
    }
  ],
  "references": [
    { "ref_id": "...", "source_tab_id": "...", "target_tab_id": "...", "columns": [ { "source_col_id": "...", "target_col_id": "..." } ] }
  ]
}
```
Upsert-only (matches by natural key, never deletes existing objects). Primary keys are expressed
purely via each column's `primary_key` flag, not a modeled index (see "When to use a unique index"
below) — `indexes` is only for genuinely additional secondary/unique indexes. Processing order:
domains → tables → columns → indexes → references.

**If no such task is found, or calling it fails, fall back to the normal flow**: create each
domain/table/column/index/reference individually following the rest of this skill, in the same
dependency order — nothing else in this skill assumes the bulk task exists.
