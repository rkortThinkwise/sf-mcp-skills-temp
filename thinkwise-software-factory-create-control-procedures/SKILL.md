---
name: thinkwise-software-factory-create-control-procedures
description: Reference guide for creating and assigning control procedures in a Thinkwise Software Factory model (code groups, business-logic variables, static/SQL assignment, dynamic model code, multi-RDBMS dialects, SQL coding guidelines). Use whenever creating, assigning, reviewing, or troubleshooting control procedures via an MCP connector with Software Factory access (e.g. sf_mcp, indicium) — before calling get_entity_definition/execute_task/execute_odata_query against control_proc, control_proc_template, code_grp, or prog_object*, and before writing or reviewing any control procedure template SQL.
---

# Creating Control Procedures in the Thinkwise Software Factory

Reference for the full control-procedure lifecycle: control procedure → template → program object
item → program object (generated stored procedure/trigger/function/view/etc., deployed to the
database, executed by the GUI or Indicium). Apply this whenever an MCP connector with Software
Factory access (`sf_mcp`, `indicium`) is used to create, assign, or inspect control procedures —
follow the connector's standard discovery→act flow; never guess entity/task/property names.

Relevant entity sets: `control_proc`, `control_proc_template`, `code_grp`, `prog_object`,
`prog_object_code`, `prog_object_item`, `prog_object_item_parmtr`, `generate_object_code`,
`branch_rdbms_type`, and — for the Task code type specifically — `task`, `tab_task`, `task_parmtr`
(see "Creating a brand-new Task" below). Relevant tasks — two distinct ones, not interchangeable (see "Actually
generating code" below): `task_generate_code_grp` (bound to `control_proc` and to the Assigning
screen's `static_assignment_overview`) creates missing `prog_object` placeholders but does not itself
produce code; `task_add_job_to_generate_object_code` (bound to `prog_object_code`/
`prog_object_overview`) is what actually queues a generation job and writes
`prog_object_generated_code`. Domain keys observed in this environment: `manage_model`/`data_modeling`
(meta-model entities) and `manage_functionality` (Functionality screen tasks) — try these directly
first (e.g. a `get_entity_definition`/`search_domain_capabilities` call against the expected domain);
only fall back to `search_capabilities`/`get_available_domains` on an
`entity_set_not_found`/`domain_not_found`-style rejection, rather than re-discovering the domain
pre-emptively every time.

## Before writing anything: confirm the plan

This skill's confirm-before-mutate obligation (see `thinkwise_software_factory_mcp_base`'s "Shared
conventions") has a concrete shape here: before the first `stage_resource` call touching
`control_proc` (or any related entity), state the plan in plain language and get the user's
explicit confirmation. At minimum, name:

- **Logic concept** — which row of the "Choosing the right logic concept" table below (Default,
  Layout, Context, Process, Trigger/Event, Task, Badge, Change detection, Handler, Subroutine, …),
  and why that one.
- **Code group** — the specific `code_grp_id` that concept maps to.
- **Target object(s)** — the table/column/task/view this will be assigned to.
- **Static vs. SQL strategy** — which one, and why (see "Static vs. SQL-typed control procedures"
  below for the trade-off to lay out).

Only stage the `control_proc` row once the user has confirmed this plan — don't treat naming a
code group or picking an assignment type as a mechanical detail to decide silently on the way to
step 3 of "Creating and assigning a control procedure" below.

## Check `branch_rdbms_type` first — before writing a single line of SQL

**Do this before writing any template, every time, no exceptions.** Query which platform(s) the model
actually targets:

`execute_odata_query` → `/branch_rdbms_type?$filter=model_id eq '<model>' and branch_id eq '<branch>'`
→ one row per enabled platform: `rdbms_type` (byte enum: `0` SQL Server, `1` DB2 iSeries, `3` Oracle,
`4` PostgreSQL) + `rdbms_name`.

**Why this has to come first, not "whenever it seems relevant":** verified live — a model enabled for
PostgreSQL only (`branch_rdbms_type` returns a single `rdbms_type = 4` row) was handed a control
procedure template written from habit in T-SQL (`getdate()`, `dateadd`/`datediff`, `select top n`).
Every `stage_resource`/`patch_resource`/`commit_resource` call along the way reported success —
**there is no dialect validation at write time.** The mistake only surfaced by explicitly reading the
generated `prog_object_generated_code` text afterward and noticing it wasn't valid PostgreSQL. Treat
"it committed" and "it generated" as proof of nothing about dialect correctness; only reading the
actual generated SQL (or checking `branch_rdbms_type` up front so the mistake can't happen) does that.

### Single row returned → single-dialect model
Write the one template in that platform's dialect — before writing any SQL, read
`references/sql_dialects.md` for the dialect/helper-function tables covering all four platforms.
`rdbms_type` is auto-filled consistently across `col`, `dom`, `prog_object`, `prog_object_code`, and
`template_prog_object_item` — no special handling needed beyond writing the right dialect in the
first place.

### More than one row returned → the shape of the work changes, not just the SQL text
1. **One dialect-specific `control_proc_template` per platform.** `control_proc_template`'s key
   (`model_id, branch_id, control_proc_id, template_id`) does **not** carry `rdbms_type` — give each
   platform's version of the logic its own `template_id` (e.g. `<name>_mssql` / `<name>_pg`, or reuse
   one `control_proc_id` with distinct `template_id`s per platform). Never write one template whose
   literal text merely happens to parse on two engines through escaping tricks — that's this exact bug
   waiting to resurface the moment one engine's syntax drifts from the other's.
2. **One `template_prog_object_item` row per `(rdbms_type, prog_object_id)`.** This junction's key
   already includes `rdbms_type`, so wire each platform's `prog_object_id` (the same logical object,
   e.g. `view_<tab_id>`, but a distinct row per `rdbms_type`) to its matching dialect-specific
   `template_id` from step 1. One assignment cannot serve two platforms.
3. **Generate and verify for every enabled `rdbms_type` separately** — both halves of "Actually
   generating code" below (`task_generate_code_grp` then `task_add_job_to_generate_object_code`) are
   addressed per `rdbms_type`. A `prog_object` existing, or generation succeeding, for one platform
   says nothing about the others. Before considering the work done: query
   `/prog_object?$filter=model_id eq '<model>' and branch_id eq '<branch>' and tab_id eq '<tab>'
   &$select=rdbms_type,generated_code_stale` and confirm a row exists with `generated_code_stale =
   false` for **every** `rdbms_type` `branch_rdbms_type` returned — then read
   `prog_object_generated_code` for each and eyeball that it's actually in the right dialect (right
   date functions, right paging syntax, right identifier quoting), not just that it generated without
   an error.
4. **Assignment can't be made "generic" to save a step.** Once there's more than one dialect, a Static
   assignment or a `template_prog_object_item` row is inherently platform-specific — resist assigning
   the same template to every platform's `prog_object_id` just because the ids/rows look interchangeable.

See "`rdbms_type` — when it's actually required" further below for the same key's role specifically in
SQL-assigned/dynamic model procedures — this section is the general version, and applies to *every*
control procedure, static or dynamic, the moment real SQL is involved.

## Static vs. SQL-typed control procedures

Every control procedure has an assignment field: **Static** (`assign_type = 0`) or **SQL**
(`assign_type = 2`).
- **Static** — pick program objects by hand, fill in `[PARMTR]` values per assignment row on the
  Assigning tab. Simple, explicit, but every new table/task needs a manual assignment. Use for
  one-off, non-repeating logic.
- **SQL (dynamic)** — a query in the control procedure decides which objects get the template and
  what each `[PARMTR]` resolves to, including duplicating a row per parameter value. Assignments
  follow automatically as the model changes. Use the moment the same template needs to apply to
  many objects, or needs to track model changes over time. Reserved in framework code for built-in
  procedures, but perfectly valid for custom logic once a pattern repeats.

Switching direction later is mechanical, not a merge: static→dynamic converts existing rows into
control-procedure code; dynamic→static materializes the query's current result as static rows.
**If it's not obvious which to start with, ask the user rather than picking one** — lay out the
trade-off (Static: explicit, one manual assignment per object, easy to reason about, more upkeep
as objects multiply; SQL: automatic fan-out that tracks model changes, but the assignment logic
itself becomes something to write and maintain) and let them choose, per "Ask, don't default" in
`thinkwise_software_factory_mcp_base`'s "Shared conventions."

### Generation strategies (SQL-typed only)
The `control_proc.strategy` enum: `delete` (0), `fully_managed` (1), `managed_via_staging_table` (2).
Docs/blog inconsistently call `fully_managed` both "Fully managed" and "Fully controlled" — same
thing.

| Strategy | Behavior | Use when |
|---|---|---|
| `delete` | Drops every previously-generated object, recreates all from scratch each run | Rarely — costs IO, risks referential-integrity errors on interdependent objects |
| `fully_managed` | Nothing auto-deleted; your SQL inserts/updates/deletes rows itself | Objects reference each other in ways the Staged diff can't resolve |
| `managed_via_staging_table` ("Staged") | Populate `#`-prefixed staging tables with desired end state; the Software Factory diffs and inserts/updates/deletes only what changed | **Default choice** for new SQL-typed procedures. Static-typed procedures always behave this way. Identities, trace columns, and calculated fields aren't settable in staging tables. |

### `control_proc_type` — separating custom logic from generated infrastructure

A separate field from `assign_type` above: `control_proc.control_proc_type` (Byte enum) marks *what
kind of control procedure this row is*, not how it's assigned. Confirmed live values:
`program_object` (`0`), `program_object_item` (`1`), `meta_definition` (`2`).

- **`program_object_item` (`1`) is hand-written logic for one specific object** — this is the actual
  custom business logic a developer wrote (e.g. "default this one column to now").
- **`program_object` (`0`) and `meta_definition` (`2`) are shared generators and framework
  infrastructure** — meta-programming control procedures that produce boilerplate for many objects at
  once (including the framework's own smoke-test/upgrade/unit-test-scaffolding machinery), or the
  wrapper templates (`_start`/`_end`) that bookend generated code. They aren't application logic
  themselves even though many generated `prog_object` rows point at them.

**When scoping "what custom logic exists to review/test/document," filter `control_proc` by
`control_proc_type eq 1` directly rather than enumerating generated `prog_object` rows for a code
type** — scanning generated objects first (e.g. "every table with a Default enabled") overcounts
wildly, since dozens of tables can share one generic generator while only a couple actually have
`program_object_item` overrides. This complements, rather than replaces, the "misleading
`control_proc_id`" reading in "Actually generating code" below — that section is for confirming which
control procedure produced an *already-generated* code section by reading its header comments;
filtering on `control_proc_type` is the faster first pass for finding candidates before you get there.

**Setting `assign_type`/`control_proc_type` when creating a `control_proc` row**: pass the enum's raw
numeric code (e.g. `0`, `1`), not its display label (`static`, `program_object_item`) — a label string
is rejected with an invalid-value error even though the same field defaults correctly to its numeric
form when left unset. If a combined add reports these fields as not-applied, a follow-up edit setting
them by numeric code resolves it.

## Choosing the right logic concept — before picking a code group

| Requirement | Concept |
|---|---|
| Fill or recalculate an entered value | Default |
| Show, hide, lock, or require fields/buttons | Layout |
| Enable tasks, reports, or detail tabs for the selected row | Context |
| Route the next step in a process flow | Process |
| Enforce integrity for all database writes | Trigger/Event, or a declarative constraint |
| Let a user or scheduler execute a command | Task |
| Show a numeric notification | Badge |
| Decide whether auto-refresh is needed | Change detection |
| Replace generated GUI/API insert/update/delete SQL | Handler |
| Reuse a database calculation or command from >1 caller | Subroutine |

For the full per-concept good-uses/avoid/best-practices, the Handler-vs-Trigger distinction, the
"choosing where a rule belongs" decision sequence, performance/security guidance, common failure
patterns, and a testing checklist by concept, read
`references/logic_concept_design_guide.md` before writing the actual business logic — the rest of
this file covers how to wire whatever concept you land on through the API, not which one to pick or
what it should contain.

## Code groups (the 24 "code types")

**Pull the live list from the `code_grp` entity set — don't assume which of the 24 groups exist or
what they're called.** This is the authoritative source; anything hardcoded here can drift stale.

Two broad families exist:
- **Business-logic groups** fire per-record/session and expose runtime `@`-prefixed variables (see
  next section).
- **Structural/generator groups** emit schema DDL or platform wrappers — just SQL text with
  `[PARMTR]` substitution, no business-logic variables.

A small illustrative subset — **non-exhaustive, and possibly stale; confirm against `code_grp`
before relying on any of these names**:

| `code_grp_id` | Family | Notes |
|---|---|---|
| `DEFAULTS` | Business-logic | Default concept |
| `HANDLERS` | Business-logic | Replaces generated insert/update/delete SQL |
| `TASKS` | Business-logic | Task code type |
| `PROCEDURES`/`FUNCTIONS`/`TABLE_VALUED_FUNCTIONS` | Business-logic | Subroutines, called "Other" in some docs |
| `VIEWS` | Structural | View SELECT code |
| `SMOKE_TESTS` | Structural | SQL Server/Oracle only; doesn't cover subroutines/handlers/tasks — those need real parameter values |
| `UPGRADE` | Structural | Migration scripts |
| `MANUAL` | Structural | Catch-all for freeform SQL not tied to any generated program-object type — brings one-off scripts (seed data, ad hoc maintenance) under the normal development-status/review/deploy lifecycle instead of running them by hand outside the Software Factory. Easy to miss. |

**A script in `UPGRADE`/`MANUAL` runs as raw SQL directly against the tables — it does not go
through those tables' Handlers.** Handlers are separate generated stored procedures invoked by the
application/API layer on insert/update/delete, not database triggers, so a seed/migration script
inserting or updating rows bypasses them entirely, with no error or warning. Any derived/computed
state a Handler would normally maintain for those rows (a cascading rollup, a computed status, an
audit stamp) has to be replicated explicitly inside the seed/migration script itself if the seeded
data depends on it — don't assume seeding a table's base columns is enough just because a Handler
exists on it.

**Two separate enablement gates exist — a table-level one and a column-level one — and NEITHER is
auto-enabled by assigning a template.** Verified wrong in an earlier version of this doc: assigning a
template to `default_absence`/`default_lead` via `template_prog_object_item` and re-running
`generate_code_grp` did *not* turn on the table's Default concept — `tab.use_defaults` stayed `false`
even though the `prog_object` rows already existed and the assignment showed up correctly in
`prog_object_item`. The work looked complete (structural wiring verified, code regenerated) but the
logic would never have actually run. Check and set **both** gates below before considering an
assignment done.

### Table-level enablement flags — verify, don't assume

Whether a code type's business logic runs *at all* for a given table is a per-table boolean on the
`tab` entity (`data_modeling` domain): `use_defaults` (Default), `use_layouts` (Layout),
`use_contexts` (Context), `use_badges` (Badge), `use_change_detection` (Change detection),
`use_insert_handlers`/`use_update_handlers`/`use_delete_handlers` (Handlers). These are the model's
real names for the "Use default/layout/context/… concept" checkboxes shown in the Software Factory
UI at the table level. An existing `prog_object` row (e.g. `default_absence`) does NOT imply this
flag is on — a table can have a Default `prog_object` (framework `defaults_start`/`defaults_end`
wrapper only, no real logic) while `use_defaults` is still `false`. Check with `execute_odata_query`
against `tab` (`$select=use_defaults,use_layouts,use_contexts,use_badges,use_change_detection,
use_insert_handlers,use_update_handlers,use_delete_handlers`) *before* declaring an assignment
complete, and enable any that are off via `stage_resource`(edit)/`patch_resource`/`commit_resource` on
the `tab` row, then re-run `generate_code_grp`.

### Per-column enablement flags — verify, don't assume

Whether a specific *column* actually participates in a code type's business-logic variables is a
separate, per-column setting, exposed as boolean flags on the `col` entity (`data_modeling` domain):
`default_input`/`default_output` (Default), `layout_input`/`layout_type_output`/`layout_mand_output`
(Layout), `context_input` (Context), `function_input` (function/task parameters). These are the
model's real names for what's shown in the Software Factory UI as "Default"/"Layout"/"Context"
checkboxes on a column. **Do not assume a newly-added or existing column has these on** — check them
with `execute_odata_query` against `col` (`$select=default_input,default_output,layout_input,...`)
*before* writing a template that references `@[col_id]`/`p_[col_id]` for that column, and again after
if the flags were off, since a control procedure referencing a column whose corresponding
input/output flag is disabled either won't have that variable generated at all or won't have the
assignment persisted back to the column. If a flag is off and the logic genuinely needs it, enable it
via `stage_resource`(edit)/`patch_resource`/`commit_resource` on the `col` row first, then write/assign
the template.

**Column flags being on is not sufficient by itself** — see the table-level flags above. A column can
have `default_input`/`default_output = true` while the table's `use_defaults = false`, in which case
the assigned logic still won't run. Check both.

## Variables — three different things, resolved at three different moments

1. **Template `[PARMTR]`** — plain text substitution at code-generation time, before compilation.
   Filled per static assignment or per SQL-assignment query row. Can substitute a column/table name,
   not just a value — a bare numeric literal works too (e.g. `dateadd(day,[days],@date_from)`).
2. **Business-logic variables** — real stored-procedure parameters (`@activated`, `@badge_value`,
   …), resolved at runtime. Set depends entirely on code type (below).
3. **Generated session variables** — session-scoped context available in *any* logic concept via
   `SESSION_CONTEXT(N'…')` (SQL Server) or `current_setting('…', true)` (PostgreSQL): `tsf_appl_id`,
   `tsf_appl_alias`, `tsf_appl_lang_id`, `tsf_global_lang_id`, `tsf_client_instance_id`, `tsf_ipv4`/
   `tsf_ipv6`, `tsf_is_public_request`, `tsf_original_login`, `tsf_use_log_session_id`, `tsf_guid`
   (deprecated).

### Business-logic variables by code type

Full per-code-type input/output variable table (Default, Layout, Context, Trigger/event, Handler,
Change detection, Badge, Process, Task), the Handler-Update PK-parameter gotcha (`@upd_[pk_col_id]`
vs. `@[pk_col_id]`), the `@cursor_from_col_id` initial-default-vs-reactive-recompute pattern, and the
dialect-dependent variable-name-prefix note (`@[col_id]` T-SQL vs. `p_[col_id]` PostgreSQL) all live in
`references/code_type_variables.md` — read it once the target code type is known, to get the exact
input/output variable names for that code type before writing the template body.

## Assignment types

- **Static**: Assigning tab → search the task/view/subject/column/other object → attach the
  template → fill `[PARMTR]` values. Check "Ignore if empty" on a parameter to drop that line
  entirely when no value is given, instead of emitting an empty string.
- **Dynamic (SQL)**: write into the staging tables backing `prog_object`, `prog_object_item`, and
  `prog_object_item_parmtr`. A parameter can fan out into multiple rows (e.g. one line per column in
  a table) — the main reason to reach for SQL assignment over static.
- `[PARMTR]` parameters auto-generate on the Parameters tab the moment they're typed into a
  template; an icon flags any still missing a value. Run **Generate parameters** if they didn't
  appear automatically.

### Static assignment via API — the actual entities involved

`control_proc_template.type_of_object`/`object_id` look like the assignment mechanism (the entity
description calls them out explicitly), but in practice static assignment for column-level logic
(Default, Layout, Context, …) is wired through a different pair of entities — verified against a
real PostgreSQL model via `sf_mcp`/`manage_functionality`:

- **`template_prog_object_item`** — the actual junction. Key: `(model_id, branch_id, rdbms_type,
  prog_object_id, prog_object_item_id)`; scalar: `control_proc_id`, `template_id`, `order_no`. One
  row = "this template contributes code, at this position, inside this generated program object"
  (e.g. `prog_object_id = 'default_absence'` is the whole table's generated Default procedure;
  framework wrapper items `defaults_start`/`defaults_end` bookend it at `order_no` 1 and 100000 —
  pick something in between). `prog_object_item_id` just needs to be unique per
  `(rdbms_type, prog_object_id)`; reusing the `template_id` as the item id is a reasonable default.
  The same low/high bookend split works for your own templates too, not just the framework's: on a
  Handler, assigning one template at a very low `order_no` (pre-mutation validation) and another at a
  very high one (post-mutation follow-up) brackets the framework's own auto-generated insert/update/
  delete statement, which sits somewhere in between at a position you don't control. **A pre-mutation
  template runs before the framework's transaction wrapper has opened one** — an unconditional
  `rollback transaction` there errors with nothing to roll back; guard it with
  `if @@trancount > 0 rollback transaction`, never call it bare. All templates assigned to the same
  `prog_object_id` concatenate into one generated procedure body, in `order_no` order — a local
  variable declared in the low-`order_no` template is visible to the high-`order_no` one on the same
  object, a legitimate way to carry a captured value (e.g. the row's pre-mutation state) from a
  pre-mutation template through to a post-mutation one.
  **Don't set the pre-mutation template's `order_no` equal to the framework's own low-end wrapper
  item (typically `1` for a Handler's `handler_start`) — verified live, a tie doesn't sort by
  intent.** When two items share an `order_no`, the platform breaks the tie alphabetically by
  `prog_object_item_id`, not by which one you meant to run first — a custom item whose id happens to
  sort before the framework wrapper's id renders *before* the generated procedure's own header and
  parameter list, referencing parameters that aren't in scope yet, and fails at deploy time (not at
  generation time). Query `prog_object_item` for the target `prog_object_id` first to see the
  framework wrapper items' actual `order_no`s (e.g. `handler_start` at `1`, `transaction_start` at
  `2`, the generated statement itself around `1000`), and pick a value strictly *between* two of them
  — e.g. `5`, safely after both `handler_start` and `transaction_start` and well before the generated
  statement — so placement never depends on an alphabetical tie-break.
  The same `<type>_<owner_id>` naming extends to tasks, not just tables — a task's own code-type
  placeholders are `task_<task_id>` (the Task code type itself), `default_<task_id>`, `layout_<task_id>`,
  `badge_<task_id>`, etc.
- **`template_prog_object_item_parmtr`** — child of the row above (same compound key +
  `prog_object_item_parmtr_id`, an auto int64). Holds `parmtr_id`/`parmtr_value` pairs: `parmtr_id`
  must match a `[bracket_token]` used literally in the template's `template_code` (case-sensitive,
  no `@` or other decoration inside the brackets — just the bare name), `parmtr_value` is the literal
  text substituted in at generation time.
- `prog_object` rows (e.g. `default_absence`) already exist once the table has ever been generated —
  check with `execute_odata_query` before assuming a `Generate code group` pass is needed.
- **A direct write (add) to `template_prog_object_item` can be rejected outright**, even though it's
  the actual junction described above. Verified live: a working fallback mirrors the Assigning screen
  instead of writing the junction table directly — find the read-only "assigning overview" row for the
  target object and control procedure (one row per `(rdbms_type, prog_object_id, control_proc_id)`,
  showing which templates are available and how many are already assigned), then invoke its bound
  "add this template's assignment" action for the specific `template_id`. That action writes the same
  `template_prog_object_item` row through a task instead of a raw insert, and succeeded where the
  direct write didn't. **This isn't universal** — confirmed live in another session that a plain
  direct add succeeded with no rejection across seven separate assignments on the same model. Try the
  direct add first; only reach for the Assigning-screen fallback above if it actually errors.
- **A direct edit to `template_prog_object_item.order_no` (to reorder an already-assigned template)
  can be rejected the same way as an add, verified live.** The working fallback is the same
  "overview" pattern as above, one level down: the assigned-templates overview row for that object
  (keyed by `rdbms_type`/`prog_object_id`/the assigning control procedure/`prog_object_item_id`, one
  row per assignment already made, distinct from the "available templates" overview used for adding)
  exposes `order_no` as a plain editable field. Edit it there instead of retrying the direct edit.
- **Generating code is two distinct tasks, not one** — see "Actually generating code" below before
  calling either. An earlier version of this doc claimed a single `task_generate_code_grp` call
  regenerates `prog_object.prog_object_generated_code`; verified live that this is wrong for a
  brand-new object.

### Actually generating code — two distinct tasks, don't conflate them

Verified live against `sf_dev_wiz_mcp`/a real model, after the simpler single-task assumption above
silently failed to produce any code for a table/view generated for the very first time this session.

1. **`task_generate_code_grp`, bound to `control_proc`** — address it with just
   `(model_id, branch_id, control_proc_id)`, no `prog_object_id` required. Use *any* control procedure
   in the target code group: your own new one, or the group's framework meta control procedure (e.g.
   `pg_views` for `VIEWS`). This materializes any missing `prog_object` placeholder row(s) for objects
   in that code group — exactly the row that doesn't exist yet for something created this session, and
   that nothing else lets you create directly (`prog_object` itself: `stage_resource`(add) → immediate
   `403`, before any field can even be set; the job-based task below: needs a `prog_object_id` that
   doesn't exist yet — chicken-and-egg). **This step alone does not generate code.** Verified: calling
   it — even twice, even bound to the framework's own `pg_views` — left `prog_object.generated_code_stale
   = true`, `prog_object_generated_code` empty, and queued nothing in `generate_object_code`. Its only
   job is to make the target object addressable for the next step. **A commit of this task can also
   report a transport-level timeout to the caller even though it completed successfully server-side** —
   verified live: the timed-out call had already created the placeholder row. Don't treat a timeout as
   proof of failure; re-query for the expected row before retrying or working around it.
2. **`task_add_job_to_generate_object_code`, bound to `prog_object_code`** (or `prog_object_overview`)
   — address it with `(model_id, branch_id, rdbms_type, prog_object_id)`, now resolvable because step 1
   created the row. This is the task that actually queues a generation job and produces code.
   - The job lands in the `generate_object_code` entity set: key `job_id`; status field
     `generate_object_code_status` (byte enum `scheduled` 0, `executing` 1, `wait_for_user` 2,
     `successful` 3, `failed` 4, `cancelled` 5, `aborted` 6, `warning` 7, `info` 8) plus a
     human-readable `generate_object_code_status_name`. No timestamp field is exposed on this entity —
     query `$orderby=job_id desc` to find the job just queued, and confirm
     `generate_object_code_status = 3` (`"Successful"`).
     **This entity set may not resolve live under that name** — verified on one connector, only a
     `generate_object_code_log` entity was found (a pure error log: `job_id`, `error_no`, `error_msg`,
     no status field at all). Don't spend time hunting for the job-status entity by name; skip straight
     to the simpler check below, which is sufficient on its own.
   - Confirm the actual output by re-reading the `prog_object` row afterward: `generated_code_stale`
     should now be `false`, and `prog_object_generated_code` should hold the real generated SQL text.
   - **`prog_object.control_proc_id` is not useful for confirming your own assignment landed** — it
     reflects whichever control procedure owns that code group's structural wrapper (the framework
     meta-procedure that also generates the `_start`/`_end` items), regardless of which procedure you
     used to trigger `task_generate_code_grp` and regardless of what your own template is assigned to.
     Verify your own logic made it in via `template_prog_object_item`/`prog_object_item`, or by reading
     the generated code text — not by checking who this field says owns the object.
   - **The same misleading field trips you up the other way round too: finding the *real*
     control procedure/template behind an existing task/object to edit its logic.** `prog_object_code`'s
     own `control_proc_id` for that row will just as often point at the framework wrapper (e.g. a
     generic per-code-type dispatcher), not the specific logic you actually want to change. The
     reliable way to find the real owner: read the generated `prog_object_generated_code` text itself —
     it contains header comments (`--control_proc_id: ...`, `--template_id: ...`,
     `--prog_object_item_id: ...`) naming the actual control procedure and template that produced that
     section of code. Look those exact names up directly in `control_proc`/`control_proc_template`
     rather than trusting any `control_proc_id` field on `prog_object`/`prog_object_code`.
3. **Order matters, and step 1 is only needed once per brand-new object.** Once a `prog_object` row
   exists — any table/view/task that's ever been generated before — skip straight to step 2 for every
   later template or assignment change. Step 1 is specifically the fix for the chicken-and-egg gap on
   an object generated for the very first time.
4. **`prog_object` is not directly writable through this API.** Don't try to hand-create or edit the
   placeholder row to work around the above — it returns `403` immediately. Always go through step 1
   instead.
4a. **Known gap, verified live: a brand-new *standalone* subroutine (`PROCEDURES`/`FUNCTIONS`/
   `TABLE_VALUED_FUNCTIONS` code group) with no existing table/view/task to hang off of may never get
   its placeholder `prog_object` materialized this way.** Unlike table/view/task-scoped code types,
   where step 1 reliably creates the missing placeholder, two different attempts both failed to
   produce one for a genuinely new subroutine: `task_generate_code_grp` bound to the group's own
   framework control procedure (the same recipe that works for `VIEWS`/`HANDLERS`/etc.), and a
   separate unbound whole-branch "generate new objects"-style task. Neither errored — they simply left
   no `prog_object` row behind to address in step 2. The control procedure, its template, and the SQL
   logic itself can still be fully authored and reviewed through the API; only the deployable object
   couldn't be materialized this way. Treat this as needing a manual generate pass in the Software
   Factory's own UI, and say so, rather than continuing to retry alternate API paths.
5. **After wiring a brand-new static assignment (`template_prog_object_item`) onto an object whose
   placeholder was bootstrapped in step 1 using a different (e.g. framework) control procedure,
   re-run step 1 again — this time bound to your *own* new control procedure — before running step
   2.** Verified live: generating right after step 1 was run only with the framework's bootstrap
   control procedure produced a `successful` status and a `generated_code_stale = false` object, but
   the generated code contained only the framework's wrapper/bookend fragments (e.g. the group's
   `_start`/`_end` items) — the newly-assigned template's own logic was silently missing. The
   assignment had been written correctly (`template_prog_object_item` looked right on inspection), but
   `prog_object_item` itself hadn't been re-synced to include it yet. Re-running step 1 bound to the
   real control procedure (not the bootstrap one) picked up the assignment; step 2 then produced the
   correct, complete code. Don't treat a `successful` status alone as proof the right logic made it in
   — read the generated text and confirm your own template's content is actually present, not just
   that generation didn't error.

### Reuse — decide in this order, before writing anything new

1. **Reuse as-is.** An existing template already does exactly what's needed — add an assignment to
   the new program object, nothing new written or reviewed.
2. **Reuse with parameters.** An existing template is right but for a different column/object — if
   it's already parameterized, add an assignment and supply the parameter values.
3. **Generalize a near-match.** A template is one specific case of a more general rule — promote the
   hardcoded parts to `[PARMTR]`s and re-assign it, including back to its original object with that
   object's own values, so both uses share one template.
4. **Write new.** Only once the above are ruled out — and even then, parameterize the object-specific
   parts so the *next* reuse doesn't require writing another one.

Search the model's existing control procedures/templates by purpose, not by object name, before
concluding nothing fits — a well-named template describes what it does.

**Reuse one template across many assignments instead of duplicating it.** If the same logic applies
to several columns/tables (e.g. "default this date column to today" on both `absence.start_date` and
`lead.converted_date`), write the template **once** with a `[PARMTR]`-style placeholder standing in
for the column's business-logic variable name:

```sql
if [date_col] is null then
    [date_col] := current_date;
end if;
```

Then create one `template_prog_object_item` row per target `(rdbms_type, prog_object_id)`, all
pointing at the same `control_proc_id`/`template_id`, and give each one its own
`template_prog_object_item_parmtr` row: `parmtr_id = 'date_col'`, `parmtr_value = 'p_start_date'` for
the absence assignment, `parmtr_value = 'p_converted_date'` for the lead assignment. Verified
production pattern for this: `refresh_after_execute_tasks` (model `62903_TASKS_AND_REPORTS`) — one
template assigned to four different task prog_objects, each supplying a different `TASK_NAME` value
via its own parameter row. Don't default to "one template per column" — that duplicates code that
should live in one place and just be re-parametrized per assignment.

**Adding new logic to an object that already has a large existing template? Add a second template
instead of editing the first in place.** A control procedure can have more than one
`control_proc_template`, each assigned to the same (or a different) `prog_object_id` at its own
`order_no` — this isn't limited to the reuse-across-assignments case above. When the change is purely
additive (new statements that don't depend on rewriting what's already there), create a new template
under the same `control_proc_id`, assign it through the normal static-assignment flow, and position it
relative to the existing item(s)' `order_no` (query `prog_object_item` first to see what's already
there, including any framework wrapper items). This avoids reading back and hand-reconstructing a long
existing `template_code` field from truncation-safe chunked reads just to safely append to it — a real
risk of introducing a transcription error into an otherwise-working script. Verified live on an
`UPGRADE`-group object seeding hundreds of lines of demo data across two existing templates; a third,
purely-additive template slotted in cleanly at a chosen `order_no` between them.

**Can't find the program object to assign to?** A table/view/task/subroutine just created doesn't
have Layout/Default/Handler/etc. program objects yet — those `prog_object` rows only exist after a
generation pass, not from creating the object itself. Run the **Generate code group** task
(`task_generate_code_grp`) for the relevant code group first — invokable from any control procedure in
that group, or directly from the Assigning screen (`static_assignment_overview`). **This creates the
missing `prog_object` placeholder(s) so you have something to assign to — it does not itself produce
`prog_object_generated_code`**; see "Actually generating code" above for the second, job-based task
that's still needed to produce real code afterward. Applies any time a target can't be found:
run this before concluding something's broken.

## Creating a brand-new Task (not just assigning logic to an existing one)

A Task is its own top-level meta-object, separate from where it's bound and from its parameters —
verified live, this order is not optional:

1. **Create the `task` row first** — keyed only by `(model_id, branch_id, task_id)`, independent of
   any table. This is the master object; nothing else about the task can exist before it does.
2. **Bind it to a table via `tab_task`** — keyed by `(model_id, branch_id, tab_id, task_id)`. Setting
   `tab_task.task_id` to a `task_id` that doesn't exist yet is rejected outright (a `403`-style error),
   even though the field presents as an ordinary editable string — it's enforcing that the `task` row
   exists first, not just validating the string shape.
3. **Add its parameters via `task_parmtr`** — keyed by `(model_id, branch_id, task_id, task_parmtr_id)`,
   a child of `task`, **not** of `tab_task`. If a parent-based add doesn't resolve a nav to `task`, fall
   back to a plain add with the full key supplied as explicit fields (see `thinkwise_datamodeling_guidelines`'s
   note on this same weak-entity fallback). Each parameter needs `dom_id` (the domain backing its
   type/control) and `mand` set explicitly; confirm `task_input`/`task_output` for output-only
   parameters rather than trusting the default.
   **A task bound to a table via `tab_task` does *not* automatically receive that table's primary key
   as an input** — unlike a Handler, which gets the row's key for free (see the code-type variable
   table above). If the task's logic needs to know which row it's acting on, add a `task_parmtr` whose
   id matches the table's primary key column (e.g. `lead_id` for a task bound to `lead`) explicitly,
   the same as any other parameter — otherwise the generated procedure has no way to identify the
   record, and this won't surface as an error until the logic is written and something is missing.
   **`task.object_name` and `task.task_description` can also reject a direct patch outright with an
   "unknown property"-style error, even set in complete isolation** — the same "looks editable in the
   staged view, rejected by the underlying write layer regardless" shape noted elsewhere for
   description-style fields (see `thinkwise_datamodeling_guidelines`'s quirks section), now confirmed
   on a non-translatable field too: `object_name` auto-derives from `task_id` (observed default:
   `<code_type_prefix>_` + `task_id`) and isn't reliably overridable this way. If a task needs a
   different generated object name than the auto-derived one, treat that as unresolved through this
   write path rather than retrying the same patch.

Only once all three exist does the usual control-procedure flow apply: run **Generate code group**
(see "Actually generating code" above) to materialize the `task_<task_id>` `prog_object` placeholder,
then assign a template to it exactly like any other code type.

## Creating and assigning a control procedure — step by step

1. **Query `branch_rdbms_type`** (see above) — know before anything else whether this is a
   single-dialect or multi-dialect model, and which platform(s) that means writing for.
2. Business Logic → Functionality (six tabs: Control procedures, Templates, Assigning, Deploy, Unit
   tests, Code review).
3. Create the control procedure: purpose-driven ID (see naming below), pick its code group, pick
   assignment type (Static to start; SQL if objects will clearly fan out). Multi-dialect: one
   `control_proc`/template set per platform (see step 1's section above) — decide this now, not after
   the first template is already written.
4. Add a template but **leave the code blank** — run **Generate code group** first so the Software
   Factory shows the real scaffold and exact parameter set for this code type, instead of guessing.
   **Caveat, verified live**: `template_code` can be enforced as a mandatory field by the write API in
   use, rejecting a true empty string. If a blank commit is rejected, use a short placeholder
   (e.g. `-- placeholder`) to get past the mandatory check, then overwrite it with the real SQL once
   the scaffold/parameter set has been seen. **`template_code` is specifically prone to the general
   "last field in a combined write can silently drop" quirk** (see
   `thinkwise_datamodeling_guidelines`) — reproduced repeatedly across separate control procedures in
   one session, always when set together with `template_description` or another field in the same
   write. **Re-tested and fixed, verified live**: the drop was an ordering artifact, not a genuine random
   bug. Setting `template_description` first and `template_code` last in the same combined
   `stage_resource`/`patch_resource` call (tested on both a fresh add and a later edit) landed both fields
   correctly every time, with no follow-up patch needed. Order the properties this way and re-read the
   result once — there's no need to isolate `template_code` into its own call.
5. Write the SQL against the generated scaffold, in the dialect(s) confirmed in step 1, save,
   regenerate to confirm valid program-object code — then actually read the generated
   `prog_object_generated_code` text, don't just trust that generation completed without error.
6. Assign it (static via Assigning tab, or SQL via staged insert against `#prog_object_item`/
   `_parmtr`). If the target object is new and missing, regenerate the code group first. Multi-dialect:
   one `template_prog_object_item` row per `(rdbms_type, prog_object_id)`.
7. Validate, optionally route through Code review, attach Unit tests.
8. Deploy: stale program objects (auto-flagged once template/assignment changed) get generated and
   pushed from the Deploy tab — all objects or just the touched ones. Multi-dialect: confirm every
   enabled `rdbms_type` generated and deployed, not just the first that succeeded.

## The comment-block header — how generated code round-trips back to a template

Documented platform behavior, not independently verified live this session: the four-line comment
block already visible when reading `prog_object_generated_code` (see point 2 of "Actually generating
code" above) is also the contract the Factory uses to identify a woven item when code is hand-edited
directly (only possible on an object that's just header/footer, per the 2026.1+ editing limits noted
earlier):

```sql
--control_proc_id:      default_fill_address
--template_id:          fill_address
--prog_object_item_id:  fill_address
--template_description: Fill in the default address
```

Rules worth knowing before touching this text by hand: keep all four lines, in this order — omitting
the first means the item isn't detected at all; omitting any other errors on save. **Never rename by
editing the header** — changing `control_proc_id`/`template_id` there doesn't rename, it creates a
*new* control procedure/template; rename through the proper task instead (see below). A
`control_proc_id` that doesn't exist yet gets created automatically from the header, which is
convenient but means a typo silently spawns a stray procedure — confirm the id first. Duplicate
`prog_object_item_id`s error on save.

## Naming guidelines

- **Control procedure IDs**: name the *purpose* the templates accomplish, not the plumbing — skip
  the code group, table name, and generic words like "default"/"task". Good:
  `calculate_discount_amount`, `revoke_user_access`, `send_order_confirmation`,
  `archive_old_tasks`. Avoid: `default_sales_order`, anything restating "this is a default/task".
- **Template names**: one template = one piece of functionality, no hidden dependency on a sibling
  template. Match the procedure's name if there's a single template; give distinct names to
  siblings otherwise. Avoid placeholders like `template1`.

## Writing dynamic model code (meta control procedures)

For writing dynamic/meta control procedures (model-generation-time SQL, tag-driven codegen,
`rdbms_type` fan-out via `branch_rdbms_type`), read `references/dynamic_model_code.md`.

## Writing SQL in the right dialect

**Before writing any SQL in a control procedure template, read `references/sql_dialects.md`** to
confirm the right dialect/helper functions for the target RDBMS. Query `branch_rdbms_type` first (see
the top of this doc) — don't assume, don't default to T-SQL out of habit. That reference file covers
all four target platforms (SQL Server, DB2 iSeries, Oracle, PostgreSQL): Thinkwise's dialect-safe
helper functions (`tsf_user()`, `tsf_send_message`, identity retrieval in a Handler) and the
null-fallback/current-timestamp/string-concat/row-limit comparison table for whatever has no
Thinkwise helper. This is also the canonical copy of this material for other Thinkwise skills
(create_view, maps_component) that need SQL-dialect guidance.

## Thinkwise SQL coding guidelines

**Before writing or reviewing any control-procedure SQL body, read `references/sql_style_guide.md`.**
It covers the most important rule first — match the codebase's existing style over any
"technically more correct" alternative — plus the general style rules (avoid `distinct`/`union`
without `all`, cursor discipline, `begin`/`end` everywhere, `tsf_send_message` not `raiserror`,
transaction pairing), the per-code-type restrictions (Triggers/Defaults/Layouts/Contexts/Processes/
Tasks/Subroutines), formatting conventions, and a full anti-pattern-vs-efficient-rewrite SQL example
(cursor-based trigger vs. the equivalent set-based `insert...select`). This is also the canonical copy
of this material for other Thinkwise skills that need SQL style guidance.

## Calculated columns — the 2026.2 `_query`-split pattern

For why calculated columns and the broader 2026.2 `<entity>_query`-split family (`dom_query`,
`tab_prefilter_query`, `col_query`, etc.) aren't safe to target with hand-written DML, and how to
detect a moved field before writing against it, read `references/calculated_columns_query_split.md`.

## Pre-flight checklist

Final scan before considering the work done — each item is a reminder, not a re-explanation; follow
the pointer for the mechanism if it's not already fresh in mind.

- Before considering a control procedure finished, confirmed with the user whether to clear its
  "in development" status — a Software Factory validation flags any control procedure left flagged
  in-development, and it's easy to forget once the SQL itself is done.
- Confirmed `branch_rdbms_type` before writing any SQL — see "Check `branch_rdbms_type` first" above.
- If the requirement didn't map to exactly one row of "Choosing the right logic concept," asked the
  user which concept(s) to cover rather than silently picking one — some requirements genuinely need
  two layers (e.g. Layout + Trigger/Handler); see `references/logic_concept_design_guide.md`'s
  "Choosing where a rule belongs."
- Preferred **Staged** over Delete/Fully managed unless there was a concrete reason not to — see
  "Generation strategies" above.
- If a new table/view/task/subroutine wasn't showing up on Assigning, ran **Generate code group**
  first — see "Can't find the program object to assign to?" above.
- Left a new template's code empty (placeholder only if blank is rejected), and set
  `template_description` before `template_code` in the same write — see step 4 of "Creating and
  assigning a control procedure" above.
- Generated code via both tasks, in order, and confirmed via `generate_object_code` /
  `prog_object.generated_code_stale` rather than trusting `committed: true` alone — see "Actually
  generating code" above.
- **Watched for silent truncation** reading back `template_code`/`prog_object_generated_code` — a long
  value can cut off with no error. Re-query in bounded chunks (multiple `substring(field,start,450) as
  cN` aliases per call; ~450 chars each, since even 500 can silently truncate) instead of trusting one
  read is complete.
- Requested `get_entity_definition` one entity at a time rather than batching several in one call —
  an entity with an `unlink_generated_object` bound task can balloon the response past the token
  limit. A `$top=1` sample read answers "what fields does this have" more cheaply when that's the
  only real question.
- Checked `col.calculated_field_type` (and the wider `_query`-split family) before writing raw DML —
  see `references/calculated_columns_query_split.md`.
- On PostgreSQL, checked every `calculated_column` expression for non-`IMMUTABLE` functions
  (`concat()`, etc.) before generating — see `references/sql_dialects.md`'s IMMUTABLE note; `42P17`
  means a function used there isn't immutable.
- If a template branches on a fixed-value (enum) domain column (a `case`/`if` keyed by e.g. a
  stage/status column), looked up that column's real per-element stored value from the domain's
  element list — the same lookup the mock-data guidance already requires for enum columns — rather
  than assuming the elements' display order maps directly to sequential integers, before hardcoding
  literals in the `case`/`if`.
- After a brand-new static assignment, re-ran `task_generate_code_grp` bound to your *own* control
  procedure (not the bootstrap one) before generating, and read the generated text back to confirm
  your logic — not just the status — actually landed. See point 5 of "Actually generating code"
  above. (`prog_object` itself stays non-writable throughout; always go through that task.)
- Remembered that generating code only updates `prog_object_generated_code` inside the model — it
  does not deploy. A "Successful" generation status also doesn't prove the SQL is valid once
  deployed: SQL Server's deferred name resolution can hide a bad/calculated-column reference until
  deploy time, surfacing as a paired "cannot find the object" + "invalid column name" error, where
  the second is the real cause.
- Layout logic is stateless per invocation — anything not re-hidden reverts to default visibility
  next call.
- Named for purpose, not plumbing — see "Naming guidelines" above.
- Knew which direction is authoritative before switching assignment type static ↔ dynamic —
  conversion is mechanical, not a merge; see "Static vs. SQL-typed control procedures" above.
- Stamped `@control_proc_id`/"Generated by control procedure" on everything a dynamic-model procedure
  creates — see `references/dynamic_model_code.md`.
- Grepped the model's existing control procedures for its established style before introducing a new
  convention — see `references/sql_style_guide.md`.
- Checked **both** the table-level and column-level enablement gates before considering a
  Default/Layout/Context/Badge/Change detection/Handler assignment complete — see "Two separate
  enablement gates" above. An existing `prog_object` row is not evidence either gate is on.
- Reused one parametrized template rather than duplicating it per column/object — see "Reuse one
  template across many assignments" above.
- If a dynamic-model procedure inserts `#msg` rows, filled in the resulting `transl_object_transl`
  text before it ships — see `thinkwise_software_factory_translation_objects`.
