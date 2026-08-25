---
name: thinkwise-software-factory-mcp-base
description: Base check to run before doing real work through any MCP connector that provides Thinkwise Software Factory access (e.g. sf_mcp, meta_dev, indicium). Confirms which connector, which application model, and which branch (model_id/branch_id) apply, since one Software Factory repository commonly hosts several application models with multiple branches each and most entities are keyed by model_id/branch_id. Use once near the start of a task that will call get_domain_definition/search_capabilities/get_entity_definition/execute_odata_query/stage_resource/stage_task/patch_resource/commit_resource — not before every individual tool call.
---

# Software Factory MCP base check

Three things have to be pinned down before real Software Factory work starts: **which connector**,
**which application model**, and **which branch**. All three are easy to get wrong silently — a
query with no explicit `model_id`/`branch_id` filter can match rows from an entirely different
model or branch in the same shared repository (see `thinkwise_datamodeling_guidelines`). Run this
check once per task, before the first substantive read/write call, not on every tool call.

## Terminology — never call it "Studio"

The Thinkwise design-time modeling tool (where a developer directly edits tables, tasks, maps,
control procedures, etc. outside of any MCP connector) is **the Software Factory** — full stop.
**Never call it "Studio"** ("open this in Studio", "check Studio", …) — that name doesn't appear in
any Thinkwise documentation, skill, or terminology this session has verified; it was a one-off,
unsourced habit from a past session and should not recur. When a manual step is needed outside what
an MCP connector can reach, say "the Software Factory" (or "the Software Factory's own UI/GUI
Modeler" for the specific screen), never "Studio" or any other unverified nickname.

## 1. Which connector

Look at the connected MCP servers that expose the Software Factory toolset (tool names like
`get_domain_definition`, `execute_odata_query`, `stage_resource` — e.g. `sf_mcp`, `meta_dev`,
`indicium`, or a differently-named instance).

- **Exactly one such connector connected** → use it, don't ask.
- **More than one**, and the user's request doesn't already name one → ask which one via
  AskUserQuestion, listing the connector names as options.
- If the user's message already names or implies the connector, use that — don't ask again.

## 2. Which application model

A repository can hold multiple application models (e.g. `RK_MERIDIAN`, `RK_SCHEDULER_TEST`,
`INSIGHTS`), each scoped by its own `model_id`. Only ask about this when the task actually touches
model-specific data (tables, tasks, screens, control procedures, process flows, etc.) — pure
metadata/discovery calls like `search_capabilities` don't need it yet.

- If the user's message, or anything earlier in this conversation, already names the application
  model (or it's unambiguous from context, e.g. only one model exists in the repository) → use
  that, don't ask.
- Otherwise, list the distinct `model_id` values available through the chosen connector (e.g. a
  grouped query on `branch`, with description) and ask the user to pick via AskUserQuestion. Skip
  the ask if that query turns up only one model.

## 3. Which branch

Once the application model is settled, the same repository query also carries `branch_id` —
each model typically has more than one branch (e.g. a mainline plus feature/test branches).

- If the user's message, or anything earlier in this conversation, already names the branch (or
  only one branch exists for the chosen model) → use that, don't ask.
- Otherwise, list all branches for the chosen `model_id` (e.g. filter that same `branch` query by
  `model_id`) and ask the user to pick via AskUserQuestion, showing every branch found.

## Once resolved

Treat the chosen connector, `model_id`, and `branch_id` as pinned for the rest of the task — carry
them into every subsequent `execute_odata_query`/`stage_resource`/`stage_task` filter and
parameter, and don't re-ask unless the user switches context to a different model, branch, or
connector.

## Shared conventions every other Software Factory skill builds on

Every `thinkwise_software_factory_*` skill (menu, tasks, process_flows, subroutines, messages,
scheduler, maps, cubes, control procedures, views, etc.) inherits these two rules instead of
restating them. If a skill's own text ever conflicts with these, follow these.

- **Confirm-before-mutate.** Before the *first* `stage_resource`/`stage_task`/`patch_resource`/
  `commit_resource` call for a piece of work, state the concrete plan in plain language — what
  will be created/changed, the key design choices behind it, and how it fits the existing model —
  and get the user's explicit confirmation. Only then start staging. This applies at whatever
  grain the work actually has: a multi-artifact goal gets one bundled plan (see
  `thinkwise_software_factory_build_planner`), a single artifact gets a short plan scoped to that
  artifact's own real design decisions (not mechanical CRUD steps like "create the row, then set
  its name"). Skills with their own dedicated propose→confirm process (build_planner, unit_tests)
  already satisfy this. **Answering clarifying sub-questions is not itself that confirmation.**
  Resolving a specific open point (e.g. via `AskUserQuestion`) is narrower than approving the whole
  plan — if anything about the plan changes afterward for any reason (new information surfaces,
  e.g. the branch's RDBMS ruling out a type you'd proposed; scope gets trimmed or expanded), re-present
  the complete, updated plan and get an explicit go-ahead on *that* before the first mutating call,
  rather than treating the earlier sub-answers as covering it.
- **Ask, don't default.** When a design choice isn't dictated by the user's own words or an
  unambiguous read of the live model, stop and ask — via AskUserQuestion or a plain question —
  rather than picking a default and iterating after the fact. This overrides any instruction
  elsewhere (including in this skill set) phrased as "default to X unless the user says
  otherwise" — treat that kind of line as *what to propose when you ask*, not as license to apply
  it silently. It's usually cheaper to ask once up front than to build something and unwind it
  later.
- **Verify unfamiliar field names with a live sample before querying on them.** Entity field naming
  isn't fully predictable from convention or from a sibling entity's schema — a table's display/label
  field doesn't always follow a `<name>_description` pattern (e.g. `menu` has no description field at
  all) and a dependent entity's own id field doesn't always mirror the parent's naming. A single
  session hit this three times (`elemnt`'s and `menu`'s guessed description field, and `task_parmtr`/
  `report_parmtr`'s guessed parameter-id field) — all cheaply resolved by pulling a live sample
  (`$top=1`, no `$select`) instead of guessing, per the token-bloat note below.

## Two hazards once real writes start (staged add/patch/commit flow)

Verified live, and confirmed repeatedly across a large batch (~100 records) of otherwise-identical
staged creates: a staging-style write API (stage a resource, patch its fields, commit it) used by these
connectors has two failure modes that produce no error at all — the write goes through, just with wrong
data — so don't trust a clean response as proof the record is correct.

- **A single patch call setting several properties at once can silently drop one of them** — not
  reliably the last one in the list, any of them, seemingly at random. After every multi-property patch,
  re-read the field values in the response and confirm each one actually took before moving on; fix any
  that didn't with a follow-up single-property patch and re-check again.
- **Patching multiple different staged resources concurrently/in parallel cross-contaminates their
  field values.** Always serialize: fully patch and commit one staged resource before starting the
  stage/patch/commit sequence for the next one, even when creating many near-identical records in a
  loop or across parallel workers.
- **A commit call can occasionally report the staged resource as not-found/expired immediately after
  a successful stage/patch in the same sequence, even though nothing else touched it.** Verified live,
  recurring across unrelated entity types in the same session (a control-procedure assignment junction
  row, and a unit-test input/output parameter row). Treat this as a transient hiccup, not a lost
  cause: re-read the target record to check whether the earlier stage/patch's change actually landed;
  if it didn't, simply redo the whole stage → patch → commit sequence from scratch rather than trying
  to resume the expired staged resource.

### Try ordering before isolating — a combined write is often fixable with one call, not several

The underlying write tools apply a multi-property list **in sequence within one call**, and a connector's
own tool description says setting one property can legitimately clear or reshape another — that's the
documented mechanism, not always a random bug. **Verified live**: a sibling skill's gotcha where patching
`process_action_type`/`task_id` on a fresh row was reported to overwrite an id field set moments earlier
was fully avoidable by putting the id-deriving fields first and the field that depends on them
(`process_action_id`) **last, in the same single call** — every one of 8 properties, including a free-text
description, landed correctly with no follow-up patch needed. Before defaulting straight to "isolate this
field into its own call, re-read, re-patch if wrong" (a 2-3+ round-trip pattern), try reordering the
properties within one combined call so the field that depends on another comes after it — verify the
single result, and only fall back to true isolation if reordering doesn't hold up on a real test.

**Re-tested and confirmed fixed by ordering alone, live, on every specific case flagged across this skill
set so far** — none needed isolation once reordered: `control_proc_template.template_code` (set after
`template_description`), `list_bar_item_description`/`list_bar_grp_description` (set after the id/target
fields), `transl_object_transl.transl_plural` (set after `transl`), and `cube_view_field.cube_area` (set
after `order_no`) — see each skill's own note for specifics. The still-open, order-independent hazard is
the general one at the top of this section (the ~100-record batch finding) — treat that one as the
genuine random case needing isolation, not a reason to isolate a specific field before testing whether
ordering alone resolves it.

### Weak/dependent entities need their full key — including the parent's — for an edit, not just their own id

Verified live: staging an edit on `list_bar_grp` by its own conceptual id (`list_bar_grp_id`) alone
failed with an `invalid_key` error naming the missing field — the entity's real key is
`(model_id, branch_id, menu_id, list_bar_grp_id)`, and the parent's `menu_id` has to be supplied even
though a read (`execute_odata_query`) filtered on `list_bar_grp_id` alone returns results fine. A
read's looser filtering isn't proof the same fields make a valid edit key. Before staging an edit on
any dependent/weak entity (one whose own id isn't globally unique without its parent), check the
error's `required_key_fields`/`missing_key_fields` if the first attempt fails, and supply every field
listed — not just the one that looks like "the" identifier.

### Batch `get_entity_definition` for a known object graph, but watch for token bloat

The metadata tool itself is built to accept several entity sets in one call specifically "to reduce
round-trips" — when a task's object graph is already known up front (e.g. building a cube needs
`cube`/`cube_field`/`cube_view`/`cube_view_field`; a menu needs `menu`/`list_bar_grp`/`list_bar_item`), ask
for all of them in one batched call rather than one entity at a time as the build reaches each one.
**Exception, verified live**: any entity carrying an `unlink_generated_object` bound task embeds the full
multi-hundred-value `type_of_object` enum verbatim in the response, and batching several such entities in
one call can balloon into a hard token-limit failure. For those entities, request one at a time, or — when
the only real question is the entity's own field names — use a live sample read (`$top=1`, no `$select`)
instead of `get_entity_definition` at all; it answers the same question far more cheaply.

### A single field's value can be truncated when read back, with no error

Verified live: reading back a large text-valued field (e.g. a control procedure template's SQL body)
through the query tool truncated the value at roughly 5000 characters, appending a truncation note —
even though the overall response was nowhere near any size limit. Changing `$select` didn't change the
cutoff; it's a per-value ceiling, not a response-size one.

**Workaround**: page through the field with an OData aggregation transformation instead of a plain
`$select` — `$apply=compute(substring(<field>, <offset>, <length>) as <alias>)&$select=<alias>`,
incrementing `<offset>` each call until the returned chunk comes back shorter than requested. The
2-argument form (`substring(<field>, <offset>)`, no explicit length) failed outright with a
query-execution error on the backend used here — always supply both the offset and a length.

Reach for this whenever a large text field (a control procedure template, a generated program
object's code, any other long text-shaped column) needs to be read back in full to verify or diff it,
not just skimmed.

### Reuse `branch_rdbms_type`/`branch_appl_lang` already resolved this session

Nearly every skill that writes SQL says "query `branch_rdbms_type` first, every time" and nearly every
skill that touches translation says "query `branch_appl_lang` to know how many languages the branch
supports" — both correct, but neither carries the same "don't re-discover what already resolved this
session" caveat this skill states for domain keys above. Apply that caveat here too: once either has been
queried for the current `model_id`/`branch_id` earlier in the same task (even by a different sibling
skill's step), reuse that result instead of re-querying — these don't change mid-task, and combining a
multi-skill build (e.g. a view, then its control procedure, then a subroutine it calls) can otherwise
re-fetch the same one or two rows three or four times over.
