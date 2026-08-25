# Known API/tooling quirks specific to a PostgreSQL migration

Each entry: symptom, root cause, and what actually worked (or the honest state if unresolved).
Complements the general write-hazard list in `thinkwise_software_factory_mcp_base` — these are
specific to enabling and porting to PostgreSQL, not general MCP-connector behavior.

## `branch_rdbms_type` is read-only through the API

**Symptom**: no `stage_resource`/`stage_task` path adds a row to `branch_rdbms_type`; the entity
reports `allow_add/update/delete = false` and has no bound tasks. Searching for an "enable platform"
task (tried `unlink_generated_object.platform` and similar) turns up nothing usable.

**Resolution**: this is a genuine gap, not a discovery failure — enabling a platform is a Project
Settings action in the Software Factory's own UI. Tell the user directly and wait for confirmation
before proceeding; don't keep searching for an API path that doesn't exist.

## Prefilter/domain/column query fields gained an `rdbms_type` dimension in 2026.2 — older guidance is stale

Some existing skill text (written before this split) states `tab_prefilter.query` has "no `rdbms_type`
dimension" and must be written portably in one slot. **This is no longer true as of 2026.2** — verified
live: `tab_prefilter_query` (a distinct child entity from `tab_prefilter`) is keyed by
`(model_id, branch_id, tab_id, tab_prefilter_id, rdbms_type)` and holds the actual `query` text per
platform, same shape as `dom_query` and `col_query`. Don't trust a skill's specific claim about which
fields are/aren't platform-split without checking `get_entity_definition` live first — this family
(`col_query`, `dom_query`, `tab_prefilter_query`, `tab_check_constraint_query`, `indx_query`,
`process_variable_query`, `report_parmtr_query`, `task_parmtr_query`, `subroutine_parmtr_query`,
`data_set_query`, `tab_query`, `unit_test_query`, `tab_data_col_query`, `dom_input_constraint_query`)
is actively growing across platform versions — see
`thinkwise_software_factory_create_control_procedures`'s `references/calculated_columns_query_split.md`
for the general detection technique.

## `execute_odata_query` truncates long text fields with no working chunked-read path

**Symptom**: reading a large `template_code`/similar field returns
`"[TRUNCATED: showing 5000 of 14393 characters]"` embedded in the response text itself (not a
tool-level "output too large" error with a file fallback — the truncation is inside the JSON payload).

**Attempted and failed**: re-querying the identical field a second time returns the identical
truncated 5000 characters (not a different window). Adding `$apply=compute(substring(template_code,
4500, 4500) as chunk2)` to the query was silently ignored — the response still returned the plain
(truncated) `template_code` field, no `chunk2` field appeared at all, meaning the connector doesn't
implement OData `$apply`/`compute` for this entity.

**No workaround found in this domain.** For a field long enough to hit this limit (a large seed-data
script is the most likely case), don't attempt a partial reconstruction — tell the user the specific
object is too large to read completely through the connector and hand that one off to be finished
directly in the Software Factory's UI (copy/paste + find-replace for the known mechanical
substitutions is enough for seed data specifically, since the content is repetitive `insert` blocks,
not novel logic).

## A brand-new (never-generated) standalone subroutine has no way to get its placeholder created via API

**Symptom**: `template_prog_object_item`/`prog_object_item` queries for the subroutine return nothing
on *any* platform (not just PostgreSQL) even though the `control_proc`/`control_proc_template` rows
with real T-SQL exist and look complete. Running `task_generate_code_grp` bound to the subroutine's own
control procedure completes without error but creates no `prog_object_item` row. Running it again bound
to the framework's own group-level control procedure (the same recipe that reliably works for views,
handlers, tasks, and triggers) *also* completes without error and *still* creates nothing.

**Resolution**: this is a real, confirmed gap specific to this one code-type shape — not a matter of
finding the right call. Write and save the PostgreSQL `control_proc_template` anyway (that part of the
API works completely normally — `allow_add`/`allow_update` are both `true` on `control_proc_template`
regardless of this gap), then tell the user this specific subroutine needs one manual "Generate" pass
in the Software Factory's UI before the new template can take effect. Anything else that `call`s this
subroutine (a Handler, a Task) will generate and look correct, but will fail at actual database deploy
time until the manual step happens — flag this dependency explicitly, don't let it look like the
migration silently succeeded.

## `task_generate_code_grp` has broader side effects than documented

**Symptom**: calling `task_generate_code_grp` bound to one control procedure (say, a trigger) to
materialize its own missing placeholder also produces stale placeholders for many unrelated objects on
the same table — badges, change-detection, generic CRUD triggers, `UPGRADE`-group objects — that
weren't the target of the call at all.

**Practical implication**: after any single `task_generate_code_grp` call, don't assume only the
object you were targeting changed state. If auditing what's stale/generated across the model, re-query
broadly (`generated_code_stale eq true` across the whole `rdbms_type`) rather than trusting that only
touched objects moved. This bulk materialization is also the direct cause of the very large stale-object
count that can trigger the MERGE/FK issue below — enabling a platform on a model with substantial
existing content routinely leaves many hundreds of objects stale simultaneously, and any subsequent
generation pass across the whole model has to process that whole batch at once.

## `MERGE` FK conflict generating a framework `UPGRADE` object at scale

See the SKILL.md section "MERGE/FK errors during generation" for the full writeup — summarized here as
a quirk-list entry: this is a Software Factory repository-internal issue (the error names a database
like `..._SF`, which holds `prog_object_item`/`prog_object_item_parmtr` meta-tables, not the deployed
application), surfaced by generating an `UPGRADE`-group object (`upgrade_pg_insert_data_in_new_tables`
specifically, but potentially any object in that group) while a very large number of PostgreSQL objects
are stale/ungenerated for the first time across the whole model. Generating base tables first, then
retrying, is the first thing to try; if it recurs, it's a product-level report to Thinkwise, not
something fixable from the modeling API.

## "Another object has the same name as the input constraint" — unresolved

See SKILL.md's "Domain/column name collisions" section for what was ruled out (`dom_input_constraint`
is empty in the affected model, so it isn't a domain-level constraint reused identically across
columns sharing a domain). The actual mechanism is still unknown as of this writing. If you hit this
message: get the full, exact validation text first (it likely names the specific object), check
whether renaming colliding domains (the fix for the sibling "same name for different objects" error)
also resolves this one as a side effect, and update this file with whatever's found — don't re-guess
from scratch.
