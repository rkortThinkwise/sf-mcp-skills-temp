---
name: thinkwise-software-factory-create-view
description: Reference guide for creating views in a Thinkwise Software Factory data model — naming, entity/column setup, domain reuse, modeling references to the view's source tables, and writing/assigning the SELECT code that backs a view. Use whenever creating, reviewing, or troubleshooting a view via an MCP connector with Software Factory access (e.g. sf_mcp, indicium), especially when converting a user's ad hoc SQL query into a view. Does not cover deployment.
---

# Creating Views in the Thinkwise Software Factory

Reference for the view lifecycle covered by this skill: `tab` (type `view`) → columns → references
→ control procedure → template → `template_prog_object_item` → generated `CREATE VIEW` program
object, generated and verified. Deployment (pushing the generated code to an actual database) is a
separate concern this skill does not cover.
Apply this whenever an MCP connector with Software Factory access (`sf_mcp`, `indicium`) is used to
create, inspect, or convert a query into a view — follow the connector's standard discovery→act flow;
never guess entity/task/property names. `tab`/`col` typically live in a `data_modeling`-style domain;
`control_proc`/`control_proc_template`/`template_prog_object_item` typically live in a
`manage_model`-style domain — try these directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session. For
the general control-procedure/template/assignment mechanics referenced throughout (code groups,
`[PARMTR]` substitution, static vs. SQL assignment, dialect differences), see the
`thinkwise_software_factory_create_control_procedures` skill — this skill only covers what's specific
to views.

## What a view is

A view is a `tab` entity with `type_of_table = view` (enum: `table` 0, `view` 1, `function` 2,
`mqt` 4). Like a table it has columns, a primary key, references, screens, tasks, security —
everything a table has — except the data is never stored; it's composed at query time from a
`SELECT` statement, so it's always current and never needs syncing.

`tab.create_view_method` picks how that `SELECT` gets defined (enum: `meta_auto` 0, `meta_custom` 1,
`template` 2):

| Method | What you write | When to use |
|---|---|---|
| **Meta Auto** (`meta_auto`) | Nothing — map each view column to a source column (`col.view_tab_id`/`col.view_col_id`), run the `task_generate_view_from_clause` bound task, the `FROM`/join logic (`tab.view_from_clause`) is derived automatically | Only a straight pull from one or a few tables joined on their modeled references, no filtering, no aggregation |
| **Meta Custom** (`meta_custom`) | Hand-edit `tab.view_from_clause`/`view_where_clause`/`view_grp_by_clause`/`view_having_clause`; the `SELECT` clause is still derived from the view's columns | Rare middle ground — custom joins/filters but still want the platform to build the select list |
| **Template** (`template`) | The entire `SELECT` statement, written as a control procedure template | Anything with real business logic: multi-table joins, aggregation, calculated fields, `UNION`, conditional logic |

**Default to Template.** In a large, mature production model inspected directly (210 views), every
single one used `create_view_method = template` — zero used Meta Auto or Meta Custom. A view earning
its keep almost always needs a join condition, a filter, or a derived column the auto-generated
`FROM` clause can't express, so teams converge on Template as the standard rather than starting with
Meta Auto and migrating later.

A fourth `type_of_table` value, `mqt` (materialized query table / snapshot), persists and
periodically refreshes the result instead of computing it live. Reach for it only once a view's
underlying query becomes a measurable, repeated performance cost — don't start there pre-emptively.

## Naming

Base rules match tables generally: singular, self-explanatory, underscore-separated, no
abbreviations, no meta-information baked into the name. View-specific points:

- **Name the view for the business question it answers, not its mechanics** — `customer_list`,
  `additional_cost`, not `join_customer_and_customer_detail`.
- **Reserve a prefix for platform-generated/dynamic-model view categories, used consistently.** The
  Thinkwise report-label pattern is the canonical example: any view feeding translated report labels
  is named with the `rpt_lbl_` prefix so the dynamic-model concept can find it. If a model has its
  own generated-view categories (audit/history, staging, …), give each one fixed prefix and document
  it — don't let it drift per developer.
- **Current official guidance is lowercase `snake_case`** (`customer_list`, not `Customer_List`).
  At least one large production model instead uses `PascalCase_With_Underscores` throughout (a
  legacy convention predating current guidelines). Follow whatever the model already uses; for a new
  model, follow the lowercase guideline. Consistency within one model matters more than which style
  is picked.
- **Name the control procedure and template after the view** so the code producing a view's data is
  trivially discoverable from the view's name. A bare match (`control_proc_id == tab_id`) is
  simplest and needs no explaining — pick one convention (bare match, or a fixed prefix like `Vw_`)
  and hold to it across the model; don't mix both.

## Confirm scope before modeling

Before creating the `tab` row, state the plan back to the user and get their confirmation — this
applies to every view being designed, not only an ad hoc-query conversion (see below). This is the
`thinkwise_software_factory_mcp_base` "Confirm-before-mutate" convention applied at this skill's own
grain; it isn't superseded by anything below about the view's own setup mechanics:

- **Source tables/columns** the view pulls from.
- **Intended grain** — what one row of the view is meant to represent.
- **Primary-key candidate** — the column(s) that define that grain.
- **`create_view_method`** — Template vs. Meta Auto vs. Meta Custom (default to Template — see
  above).

**Flag, don't silently decide, if the request is a poor fit for a plain view** — heavy aggregation
over a large table, queried by dozens of other things, on source data that barely changes. A
snapshot (`type_of_table = mqt`) or a scheduled denormalized table is often the better trade; raise
it with the user rather than defaulting to a view.

## Setting up a view — the basics

1. **Data model → Tables → New table.** Set `type_of_table = view`, pick `create_view_method`
   (default `template`).
2. **Model the columns before writing any code.** For Template views, the control procedure's
   `SELECT` list must match the view's modeled column list, in the same order — model columns first,
   then write `SELECT` to match, not the other way around.
   - **Column order**: same 10-per-step convention as regular tables (`col.order_no` 10, 20, 30, …),
     leaving gaps for later insertions.
   - **Primary key**: pick the column(s) that define the view's actual **grain** (the level of
     detail one row represents) — usually the query's `GROUP BY`/`DISTINCT` columns, or the source
     table's PK for a row-for-row pull. If the grain isn't obvious from the request, ask the user what
     one row of the view is meant to represent before picking the columns/primary key. Getting the
     grain wrong (a PK that isn't actually unique per row) is the most common view bug — nothing
     enforces uniqueness on a view by default, so verify manually against real data
     (`GROUP BY <candidate PK> HAVING COUNT(*) > 1`) before shipping.
   - **Reuse existing domains** for every column representing the same concept as an existing column
     elsewhere in the model — see next section.
3. **References are part of modeling the view, not an optional follow-up.** For every column on the
   view that is FK-shaped (its value is meant to match another table's primary key — e.g. an
   `employee_id` column, or an `absence_id`-style pointer), add the `ref`/`ref_col` at this point,
   before writing the query. They are never implied by the view's `FROM` clause; Meta Auto/Meta
   Custom infer references from mapped source columns, but Template gives none for free — skip this
   step on a Template view and the column silently has no look-up, no detail grid, and no
   auto-derived OData/Indicium navigation, even though the generated `SELECT` still returns the right
   data.
   - See the `thinkwise_datamodeling_guidelines` skill's "References (foreign keys)" section for the
     full look-up-vs-detail explanation. The one point specific to views, repeated here because it's
     easy to get backwards: **the direction is reversed from a normal table-to-table FK.** For a view
     column pointing at a real table's primary key, `source_tab_id` = the real table, `target_tab_id`
     = the view — not the other way around. Getting this backwards makes the reference render as a
     detail grid on the view instead of the look-up you almost always want there.

     Example — view `employee_absence_overview`, column `employee_id` pointing at `employee`:
     ```
     ref:      source_tab_id = employee,   target_tab_id = employee_absence_overview,
               check_ref = false,   show_detail = false
     ref_col:  source_col_id = employee_id,   target_col_id = employee_id
     ```
     `check_ref = false` because a view carries no physical FK constraint; `show_detail = false`
     because a detail grid on the real table listing view rows is rarely useful. If two or more view
     columns point at the same target table, set `ref_add` on each (e.g. `"most_recent"` /
     `"upcoming"`) so their auto-derived `ref_id`s don't collide.
4. Write and assign the actual query (below).
5. Regenerate and verify the generated code (below).

**The new view and its columns need translating, same as any table.** `tab`/`col` created here get
the platform's usual bracket-placeholder translation until filled in — see
`thinkwise_software_factory_translation_objects` for the detection query and field reference. Do
this after the column list is finalized (step 2) so it isn't repeated every time a column gets
renamed while the view is still being modeled.

## Reusing columns and domains

A view column is not "read-only, so it doesn't matter" — every domain decision made for the source
tables should carry through:

- **Match the domain (`col.dom_id`) of the source column exactly**, not just the data type. If
  `customer.customer_code` uses domain `Customer_Code`, the corresponding view column surfacing the
  customer code should use the same `Customer_Code` domain — not a fresh domain with the same
  `NVARCHAR(20)` shape. This keeps default UI controls, translations, input constraints, and
  domain-level validation consistent everywhere the value appears. Verified in a real production
  view: `Customer_Code`, `Customer_Group`, `Currency`, `Variety_No` and other columns all reuse the
  exact same domain IDs as their source tables; generic domains like a date domain or a datetime
  domain get reused wherever a date/datetime is surfaced, rather than each view minting its own.
- **Only introduce a new domain for a genuinely new business concept** — a computed/derived value
  with no 1:1 existing column (a calculated total, a concatenated display string, a derived flag).
  Don't create a "view-only" duplicate of an existing domain out of convenience.
- **Aggregations/calculations should still reuse the base domain where the semantics match** (a
  summed amount column can use the same currency/amount domain as the column being summed) — only
  diverge if the aggregation changes the meaning (a `COUNT` result is a plain integer, not whatever
  domain the counted column used).
- **FK-shaped columns inside a view keep the same column name as the table they reference**, exactly
  like normal FK columns — this is what lets Indicium/OData auto-derive the relationship and keeps
  look-ups working without extra configuration.

## Writing and assigning code to a view (Template method)

This is the verified, concrete object chain behind every Template view in a production model, traced
end-to-end via the Software Factory metadata API:

`tab` (view) → generation produces a structural `prog_object` named `view_<tab_id>`, owned by the
framework's `VIEWS`-code-group meta control procedure → your own `control_proc` supplies the actual
`SELECT` as a fragment woven into that `prog_object` via a `template_prog_object_item` row.

1. **Query `branch_rdbms_type`** (`/branch_rdbms_type?$filter=model_id eq '<model>' and branch_id eq
   '<branch>'` — see the control-procedures skill's "Check `branch_rdbms_type` first" section) before
   writing anything. One row → write the view's `SELECT` in that platform's dialect. More than one row
   → this view needs one dialect-specific `control_proc_template` per platform and one
   `template_prog_object_item` per `(rdbms_type, prog_object_id)` — decide this now, it changes steps 3
   and 5 below, not just the SQL text in step 4. Skipping this is exactly how a view ends up with
   `getdate()`/`dateadd`/`select top n` generated against a PostgreSQL-only model, or `limit`/`age()`
   generated against a SQL-Server-only one — every write along the way still reports success either
   way, so this has to be checked up front, not diagnosed after the fact.
2. **Model the view's columns first** (above) — order, primary key, domains. The `SELECT` written in
   step 5 must produce exactly this column list, in this order, with matching aliases.
3. **Make sure the view's structural program object exists.** A brand-new view has no generated
   program objects yet. Run **Generate code group** for the `VIEWS` code group (`task_generate_code_grp`,
   bound to `control_proc`, addressable by just `(model_id, branch_id, control_proc_id)` — use any
   control procedure already in `VIEWS`, e.g. the framework's own `pg_views`, since your own control
   procedure for this view doesn't exist yet at this point) once so the framework's `view_<tab_id>`
   program object shell exists — nothing can be attached to it before that. **This only creates the
   placeholder row — it does not generate any code yet**; see step 6 and the
   `thinkwise_software_factory_create_control_procedures` skill's "Actually generating code" section
   for why that's a second, separate task. Multi-dialect models: this creates one `prog_object` row per
   enabled `rdbms_type` for the same `tab_id` — expect (and plan to fill) all of them, not just one.
4. **Business Logic → Functionality → Control procedures → New** (`control_proc` entity):
   - `code_grp_id = VIEWS`
   - `control_proc_type = program_object_item` (the code is a *fragment* woven into the generated
     `CREATE VIEW` object, not a standalone object)
   - `assign_type = static` (one view = one hand-picked assignment; SQL/dynamic assignment is for
     framework-internal or genuinely repeating patterns, not a one-off view query)
   - ID: match the view name, or the model's chosen prefix convention (above). Multi-dialect: either
     one `control_proc` with one `control_proc_template`/`template_id` per platform, or a distinct
     `control_proc` per platform — pick one convention and hold to it across the model, same as any
     other naming decision here.
5. **Add a `control_proc_template`, but leave `template_code` blank at first** and regenerate — this
   surfaces the real scaffold instead of guessing at it. **Caveat, verified live**: `template_code`
   can be enforced as mandatory by the write API in use, rejecting a true empty string — if so, use a
   short placeholder (e.g. `-- placeholder`) to get past the check, then overwrite it with the real
   `SELECT` once the scaffold has been seen. Then write the full `SELECT`, **in the
   dialect(s) confirmed in step 1**:
   - Column list matches the view's modeled columns, in order, aliased to the exact `col_id`s.
   - Grep a sibling view's template in the same model — and the same `rdbms_type`, if multi-dialect —
     for join style, alias conventions, paging syntax (`limit` vs. `top`), date/time functions, and
     comment-header style before introducing a new one — consistency beats individual preference (see
     the style-continuity rule in the control-procedures skill).
   - Follow the general Thinkwise SQL guidelines (see the control-procedures skill): lowercase
     keywords, explicit column lists, no `SELECT *`, comment non-obvious joins/filters, avoid
     `DISTINCT` where a `GROUP BY` does the same job more explicitly.

   ```sql
   -- Example shape, adapted from a real production view template (PostgreSQL dialect)
   select  c.customer_code
          ,cd.company_no
          ,c.customer_short_name
          ,c.customer_group
          ,cd.customer_type_id
   from    customer c
   join    customer_detail cd
       on  cd.customer_code = c.customer_code
   where   c.active = 1
   ```
6. **Wire the template into the generated view object.** Two equivalent routes (same mechanism as
   any other static assignment — see the control-procedures skill's "Static assignment via API"
   section for the full entity/field breakdown):
   - **UI**: Functionality → Assigning tab → find the view's program object → attach the template.
   - **API/dynamic**: insert a `template_prog_object_item` row — `prog_object_id = 'view_<tab_id>'`,
     `control_proc_id`/`template_id` = the new control procedure/template, `order_no = 10` (leaves
     room to insert more fragments later, e.g. a second comment block or a `UNION` branch).
     Multi-dialect: one such row **per `rdbms_type`**, each pointing `prog_object_id`'s `rdbms_type`
     at the matching dialect-specific `template_id` from step 5 — never point two platforms' rows at
     the same template.
7. **Generate the actual code — two distinct tasks, don't conflate them.** See the
   `thinkwise_software_factory_create_control_procedures` skill's "Actually generating code" section
   for the full generate-code-group vs. generate-object-code walkthrough, including the
   placeholder/re-run gotcha (a brand-new static assignment needs `task_generate_code_grp` re-run
   against *your* control procedure, not just the framework one from step 3, or the generated body
   silently keeps only the wrapper) — that mechanic is generic to any `program_object_item` control
   procedure, not specific to views. What's specific to views:
   - The generated program object is always named `view_<tab_id>`. Target
     `task_add_job_to_generate_object_code`'s `prog_object_code` key at
     `(model_id, branch_id, rdbms_type, prog_object_id = 'view_<tab_id>')`.
   - After generation, re-read the `prog_object` row and sanity-check
     `prog_object_generated_code` against intent (column list, joins, filters match what was modeled)
     **and against the dialect confirmed in step 1** (right date/time functions, paging clause,
     identifier quoting) — a clean generate says nothing about dialect correctness by itself. If the
     view replaces an ad hoc query (below), diff the generated SQL's logic against that original query
     directly.
   - **Multi-dialect: repeat generation and the read-back for every `rdbms_type` in
     `branch_rdbms_type`**, not just the first one that works. Query
     `/prog_object?$filter=... and tab_id eq '<tab>'&$select=rdbms_type,generated_code_stale` and
     confirm `generated_code_stale = false` for all of them before considering the view done.
   - This is as far as this skill goes — deploying the generated code to a database is a separate step
     outside its scope, and `prog_object` isn't directly writable through this API either — don't
     assume a successful generate means deploy is also possible; confirm with the user first.
8. **Validate / code review as normal** — move `development_status` through review, attach unit
   tests if the view feeds anything business-critical (a report, a financial calculation, a process
   flow decision).

## Recursive/hierarchical (explosion) views

For a self-referencing hierarchy (a bill-of-materials explosion, an org chart, a category tree), the
Template `SELECT` is almost always a recursive CTE — this brings SQL-Server-specific gotchas around
query hints and type matching between the anchor and recursive members. See
`references/recursive_views.md` for the full detail before writing one.

## Converting a user's ad hoc query into a view

Use when a user hands over a working SQL query (from SSMS, a report tool, a BI dashboard) and asks
for it to become a proper view. See `references/adhoc_query_conversion.md` for the full workflow —
it follows "Confirm scope before modeling", "Setting up a view — the basics", and "Writing and
assigning code to a view" above step-for-step, with a handful of conversion-specific differences
(deriving grain/PK from the query, confirming source names against the live model, diffing the
result against the original query).

## Pre-flight checklist

- **Query `branch_rdbms_type` before writing a single line of the `SELECT`** — one row, one dialect;
  more than one row, one dialect-specific template + one `template_prog_object_item` per `rdbms_type`,
  and verify generation for every one of them, not just the first that works. Verified live: writing
  the wrong dialect (T-SQL against a PostgreSQL-only model) commits and "generates" without any error
  at any step — the mistake only shows up by reading the actual generated SQL text.
- Model columns (order, PK, domains) before writing the `SELECT` — the template must match the
  column list, not the reverse.
- Default to `create_view_method = template` unless the pull is genuinely a trivial single/few-table
  join with no filter or aggregation.
- Set the view's own `tab.icon_id` to a suitable icon (the view is a `tab` row like any table) — a
  view built as a business work queue should get the work concept (e.g. "orders to release" →
  order/check), not a generic view/database icon. Follow `thinkwise_software_factory_icons`.
- Reuse the source column's exact `dom_id` for every view column that isn't a genuinely new derived
  concept.
- Verify the chosen primary key is actually unique per row against real data — a view enforces
  nothing on its own.
- Model a `ref`/`ref_col` for every FK-shaped column before writing the template — never leave it as
  "just a column with matching values." Remember the direction reversal for views: `source_tab_id` =
  the real table, `target_tab_id` = the view (opposite of a normal table-to-table FK). Getting this
  backwards renders as a detail grid on the view instead of the look-up you want.
- New view not showing up to attach code to? Run **Generate code group** (`task_generate_code_grp`,
  bound to `control_proc`) for `VIEWS` first — but that only creates the placeholder `prog_object`,
  it doesn't produce code by itself.
- Leave a new template's code empty and generate the code group first — don't hand-guess the
  scaffold. If the write API rejects a true-empty `template_code` as mandatory, use a short
  placeholder comment instead of blank.
- **After wiring a brand-new `template_prog_object_item` assignment, re-run `task_generate_code_grp`
  bound to your own control procedure (not the framework one used to bootstrap the placeholder)
  before generating** — otherwise generation can report success while the view body silently contains
  only the framework's wrapper fragments, missing your own `SELECT`. Read the generated code back and
  confirm your own query is actually present, not just that the status was "Successful".
- Actually producing/refreshing the view's SQL is a **second**, separate task —
  `task_add_job_to_generate_object_code` bound to `prog_object_code`. Confirm it actually ran via the
  `generate_object_code` entity (`generate_object_code_status = 3`/`"Successful"`, sorted
  `$orderby=job_id desc`) and `prog_object.generated_code_stale = false`, not just a `committed: true`
  from the task commit.
- `prog_object` is not directly writable (`403` on `stage_resource` add) — never try to hand-create or
  patch it; always go through `task_generate_code_grp`.
- Generating code stays inside the model (`prog_object_generated_code`) — it is not the same as
  deploying to a live database, and this connector may not have rights for that separate step. Don't
  offer to execute/deploy without checking with the user first.
- Grep a sibling view's template for join/alias/comment style before introducing a new convention.
- When converting a user's query: confirm every source table/column against the live model (don't
  trust the raw SQL's names), and diff the view's output against the original query as the
  acceptance test.
- Pick one control-procedure naming convention for views (bare match vs. a fixed prefix) and hold to
  it across the model.
- The view and its columns are translatable objects with the same bracket-placeholder default as any
  table/column — fill them in via `thinkwise_software_factory_translation_objects` once the column
  list is stable. **Before calling the view done, run the translation completeness gate from
  `thinkwise_datamodeling_guidelines`'s "Translating new objects" section** — don't rely on
  remembering to translate each column as it's created; a real session translated the view itself and
  still left every column at its bracket-placeholder text.
- **Writing a recursive/self-referencing view?** Read "Recursive/hierarchical (explosion) views"
  above first — SQL Server rejects `OPTION (MAXRECURSION n)` inside a view body outright, and
  requires exact type matches (cast domain-backed columns, and re-cast accumulating arithmetic every
  step) between the CTE's anchor and recursive members.
