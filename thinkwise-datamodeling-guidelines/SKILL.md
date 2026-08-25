---
name: thinkwise-datamodeling-guidelines
description: Reference guidelines for designing and naming domains, tables, and columns in a Thinkwise Software Factory data model — naming conventions, entity classification, column order, data types, references, and menu placement. Use whenever designing or reviewing a data model, or making sf_mcp calls that create or inspect domains, entities, columns, or references, to check names, types, and structure against these conventions before proposing or executing changes.
---

# Thinkwise Data Modeling Guidelines — Domains, Tables, and Columns

Reference conventions for the Thinkwise Software Factory data model, compiled from the official
Thinkwise data modeling guidelines (docs.thinkwisesoftware.com) and Thinkwise Community articles.
Apply these when designing a new data model, reviewing an existing one, or using sf_mcp tools that
touch domains, entities, or columns.

## Bulk-importing a whole data model in one call — check for a custom task first

Some models have a **custom-built** task (not a standard feature — may or may not exist in the model
you're working with) that upserts a whole batch of domains/tables/columns/indexes/references from one
JSON payload, instead of creating each object individually via the flow described in the rest of this
skill. See `references/bulk_import_data_model.md` for how to check whether it exists, confirm its
shape, and fall back to the normal per-object flow if it doesn't.

## General naming rules (apply everywhere)
- **Singular, lowercase names** — never plural, never mixed case.
- **Self-explanatory names** — a reader shouldn't need extra context to understand what it represents.
- **Split into subnames with underscores** (`sales_order_line`, not `salesorderline` or `SalesOrderLine`).
- **Avoid abbreviations** — spell words out fully, except where a platform name-length limit forces it, or for the conventional exceptions `id` and `no`.
- **No meta-information in the name** — don't encode data type, length, or table membership into the name itself.
- **Prefer reusing an existing, already-reviewed naming word component over inventing a new one when
  an equivalent term already exists.** Every new distinct word introduced into a table/column/domain
  name enters its own review step (a Software Factory validation tracks unreviewed naming components)
  — reusing vocabulary already used elsewhere in the model, when the meaning genuinely matches, avoids
  growing that backlog for words that already mean the same thing. This is the naming-level version of
  the label-reuse rule under "Form & grid groups" below.
- **Never name a column or domain after a SQL reserved/keyword-adjacent word** — `level`, `year`,
  `order`, `group`, `date`, `time`, `table`, `view`, `index`, `key`, `value`, `user`, `values`,
  `check`, `default` — even when the target RDBMS happens to tolerate it unquoted. These names get
  woven directly into generated SQL identifiers, and hitting one live (both `level` as a column and
  `year` as a domain, in the same model) surfaced the problem only once hand-written SQL referencing
  them was deployed, not at generation time. If a column/domain already has one of these names, use
  the dedicated rename task (for columns/domains, not a delete-and-recreate) to fix it — but the
  rename task only updates model metadata and structural DDL; it does **not** rewrite any
  hand-authored control-procedure/view/task template text that already referenced the old name by
  string, so grep every template referencing the old identifier and update it by hand afterward, then
  regenerate. **It also leaves the old id's translation object behind as an orphan** — the renamed
  column/domain gets a fresh `transl_object`/`transl_object_transl` under its new id, but the old id's
  rows aren't deleted, just no longer reachable from anything live. Run
  `task_delete_unused_transl_objects` (bound to `branch_appl_lang` — see
  `thinkwise_software_factory_translation_objects`) afterward as a branch-wide cleanup, rather than
  leaving stale rows to accumulate.

## Language of names
Every name (tables, columns, domains, domain elements) is written in one consistent human language —
this skill's own examples are English, but the actual language is whatever the model already uses.
- **Expanding an existing data model**: match the language already in use. Check existing table/column/domain
  names before adding anything — if the model is in Dutch (`werknemer`, `verkooporder`), new additions must
  be Dutch too, not English, even though this document's examples are in English. Don't mix languages within
  one model.
- **Starting a new data model from scratch** (no existing tables/domains to infer from): **ask the user what
  language to model in** before naming anything, unless they've already stated it (e.g. they wrote the request
  in a specific language, or named entities themselves in it) — in that case just follow their input instead of
  asking a redundant question.

## When a guideline conflicts with an existing model's own established convention

The rules in this document describe the ideal. A real, already-existing model sometimes doesn't
follow it consistently — e.g. every table already shares one generic domain for surrogate keys
instead of a per-entity domain (see "Never create/use a bare `id` domain" below), or the model has
never used diagrams, or never tuned per-column sort/search/filter/grouping settings anywhere. This is
the same shape of decision as the language rule just above, generalized: **when expanding an existing
model, matching its real, pervasive convention for new/expanded objects is usually more valuable than
introducing a new, inconsistent convention that only the newest objects follow** — even when that real
convention falls short of this document's stated ideal. Silently "fixing" only the new objects creates
a model that's inconsistent with itself, which is its own cost.

This isn't a license to ignore every guideline below — it applies specifically when a convention is
genuinely **pervasive** across the existing model (essentially every comparable table/column already
does it the same way), not when only one or two prior objects happen to deviate. When it applies,
**say so explicitly** rather than silently picking a side — note the deviation from the ideal (or the
deviation from this document) so the tradeoff is visible to whoever reviews the change, instead of
letting it pass unremarked either way.

## Tables (entities)
Classify every table into one of four types before modeling its columns; if a table doesn't cleanly fit one, that's a signal to restructure rather than just rename:
1. **Strong entities** — exist independently, single-column primary key with no foreign keys as part of it (e.g. `sales_order`).
2. **Weak entities** — can't exist without a parent (e.g. `sales_order_line`). Name = parent entity name + qualifying addition. **The foreign key to the parent is part of the primary key, and comes first (topmost) in it** — followed by the entity's own discriminating column(s). See "Column order" below for exactly where an identity column fits into that sequence.
3. **Link tables** — resolve many-to-many relationships. **The primary key is the composite of the foreign keys to the two linked tables — no separate identity column is needed**, since the pairing itself is the row's identity. **Name** = `<table1>_<table2>`, and the **primary key column order follows the same order as the name**: for a link table `employee_employee_function`, the primary key is `employee_id` first, then `employee_function_id` second. (If the association needs to repeat over time — e.g. it carries a historized start/end date range, so the same pair of foreign keys can legitimately appear in more than one row — it's no longer a pure link table; it becomes a weak entity of one side with its own identity column added as the last primary key column, per rule 2 and "Column order" below.)
4. **Inheritance tables** — implement a 1:1 "is-a" relationship with a parent; named for the specialization.

A built-in validation (`tsf_guidelines_categorize_entities`) flags tables that don't qualify as any of the four.

### Table description

**`tab_description` should say more than the table's own name restated in words.** A description
that just spells out the title (`sales_order` → "Sales order") tells a reader nothing they couldn't
already see from the name itself. Write what the table is actually *for* — the role it plays in the
model, what business process or concept it captures, and anything a reader wouldn't guess from the
name alone (e.g. `sales_order` → "A customer's confirmed order for one or more products, used to
drive fulfillment and invoicing"). This applies whenever a table is created or reviewed, the same as
naming and classification above.

### Icon

Set `tab.icon_id` to a suitable icon from the repository as part of creating the table — it's the
table's own icon wherever it's shown (menu item, document, tree/detail navigation), not a cosmetic
extra to skip. Follow `thinkwise_software_factory_icons` for how to search/reuse the repository and
what to do if nothing fits (ask the user, don't guess or leave it blank). A view used as a business work
queue should get the work concept (e.g. "orders to release" → order/check), not a generic table/view
icon — see that skill's subject-icon guidance.

### Exposing new tables on a menu

Creating a table is only half of making it usable. A **strong entity** (rule 1) — or the parent side of
an **inheritance** relationship (rule 4) — is a candidate **top-level subject**: something a user should
be able to open directly from the menu, not just reach by drilling into a parent's detail grid. **Weak
entities (rule 2) and link tables (rule 3) are usually not** top-level subjects — they exist to be detail
rows or associations under something else, and defaulting them onto the menu clutters it with entries
nobody opens directly. Treat that as a strong default, not an absolute: occasionally a weak entity is
genuinely browsed standalone, and that's a judgment call, not a rule violation.

**Don't decide silently which candidates become menu items.** After creating or reviewing a batch of new
tables, present every candidate top-level subject as a **multi-select checkbox-style question** (one
option per table, so the user can tick which get added and leave the rest unticked — e.g. because
they're only ever reached through a parent's screen, or aren't ready for end users yet) rather than
adding all of them, or guessing which "obviously" belong.

For the ones the user picks, follow the `thinkwise_software_factory_menu` skill for the actual
menu/group/item work — including its own golden rule to confirm menu type and group placement, and to
create a new menu (rather than assuming an existing one fits) if the model doesn't have one yet for the
relevant platform.

## Columns
- Same general rules: lowercase, singular, self-explanatory, underscore-separated.
- **Don't prefix non-key columns with the table name** — redundant since the table context is already known.
- **Primary key**: name = `<table_name>_id`.
- **Foreign key**: must have the exact same name as the parent table's primary key column — this is also what lets OData/Indicium auto-derive relationship names.
- Prefer `INT` identity for surrogate primary keys; use `BIGINT` only for tables expected to exceed ~2 billion rows. Only non-FK primary key columns should be identity columns.
- **One fact per column, atomic values only** — don't pack multiple values into a delimited string.
- **Prefer NOT NULL with a sensible default** over nullable columns, unless the absence of a value is itself a meaningful business state.
- **A column's default value must match its data type and, if backed by a domain with elements, be a
  valid element (or within the domain's min/max range for a ranged domain).** This is checked by a
  Software Factory validation, not enforced at write time — `stage_resource`/`patch_resource` will
  commit a default value that's the wrong type or out of range without complaint, and it only surfaces
  later as a validation error. Double-check a newly-set default against its column's actual domain/type
  before considering the column done, including for a data-migration script's own column defaults.
- **Before finalizing a new column's name, check its translated label doesn't collide with another
  column's label already in use on the same table** — two columns with different ids but identical
  rendered captions is flagged by a Software Factory validation and confuses a user reading the
  form/grid.
- **A column can be calculated/expression-backed (`calculated_field_type` non-zero) instead of physically stored** — it reads back identically to a real column on a normal query, with no visible difference. See "Calculated columns" below for the different kinds and when (rarely) to reach for one; this matters most when writing hand-written SQL against the table directly (control procedures, migrations) — see `thinkwise_software_factory_create_control_procedures`'s "Calculated columns are not physical" note before including such a column in an insert/update/merge statement.

## Data sensitivity classification

Every column carries a data-sensitivity/privacy classification, and a Software Factory validation
flags any column left at its undecided default — more strongly for columns the platform can already
infer are *likely* sensitive from their name/domain. Decide it deliberately when creating a column,
the same as its domain/type: for an obviously personal or confidential field (name, email address,
national id, financial account, health data) set the classification explicitly rather than leaving it
unset; for a column where sensitivity genuinely isn't obvious, ask the user rather than guessing — this
has real compliance consequences, not just a lint warning. Confirm the exact field/enum name live via
the column entity's own metadata before writing to it.

## Calculated columns

**Default to a real, physically stored column (`calculated_field_type = none`) and only reach for a
calculated column when the value genuinely cannot be a normal one** — it must always be perfectly
derived from other data that can change independently, with no acceptable point where it could
instead just be set once (an insert/update-time Default control procedure, or plain application
logic). A calculated column is a special-case tool for that narrow situation, not a stylistic
alternative to writing normal columns — most of a table's values belong in real columns, per "One
fact per column" and "Prefer NOT NULL..." above.

`col.calculated_field_type` has four values, verified live against real models:

| Value | Meaning | What it actually is |
|---|---|---|
| `none` (0) | Real column | Ordinary, independently writable — the default. |
| `expression` (1) | Cross-row/cross-table formula | `calculated_field_query` is evaluated as a correlated subquery, referencing the current row via alias `t1` (e.g. `t1.project_id`) — free to join or subquery into *other* tables. |
| `calculated_column` (2) | Same-row computed column | `calculated_field_query` is the literal body of a native `AS (<expr>) [PERSISTED]` computed column — may reference only *other columns already on the same row*, no `t1.` prefix, no reaching into other tables. |
| `calculated_column_function` (3) | Function-backed computed column | **Unconfirmed** — no live example found in any model checked. `col.function_input` marks which column(s) feed a shared scalar function as parameters. Verify its actual generated shape in a real model before relying on it, rather than assuming this description is complete. |

**PostgreSQL-specific pitfall, verified live**: a `calculated_column`'s expression compiles to a
native generated/persisted column on PostgreSQL, which requires every function used to be
`IMMUTABLE` — `concat()` is `STABLE`, not `IMMUTABLE`, and fails with error `42P17`. See
`thinkwise_software_factory_create_control_procedures`'s `references/sql_dialects.md`
("PostgreSQL generated/calculated columns require IMMUTABLE functions") for the fix and the
NULL-handling difference between `||` and `concat()`. This does **not** apply to `expression`-type
columns below — those are correlated subqueries, not stored generated columns.

**A calculated column still needs `col.dom_id` set, the same as any physical column, verified
live** — even though its value comes from `calculated_field_query` rather than being written
directly, the domain is still mandatory (it's what drives the column's data type/display). This is
easy to miss when a table's other domains are all entity-specific. When a calculated column used as a
table's look-up display column (see "Look-up display column" below — and give it a descriptive
name there, never the generic `lookup`) doesn't map naturally onto any domain you're already
creating for the table, consider one small, genuinely reusable domain for that purpose (a generic
display-label string type) shared across tables' calculated display columns, rather than inventing a
one-off domain per table.

### When to use which

- **`none`** — the default, always, unless one of the below genuinely applies.
- **`calculated_column`** — when the formula only touches other columns already on the same row:
  arithmetic, `concat`, `cast`/`case`, date differences, bitwise flips, hashes. Confirmed live
  examples: `active = ~archived`; `duration_seconds = datediff(second, start_date_time, finish_date_time) PERSISTED`;
  `full_name = concat(first_name, ' ', last_name)` (SQL-Server-shaped; if PostgreSQL is a target
  platform, write this as `coalesce(first_name,'') || ' ' || coalesce(last_name,'')` instead — see
  the PostgreSQL caveat above). Decide `PERSISTED` vs. not based on read/write balance — see
  Performance below.
- **`expression`** — only when the value genuinely needs to reach outside the row: a join/subquery
  to another table, a translation fallback, a session-context-dependent value (current
  language/user). This is the more expensive option (see Performance below), so don't reach for it
  when `calculated_column` would do.
- **`calculated_column_function`** — only for logic complex/reusable enough to justify a shared
  function definition instead of an inline expression, and only after confirming its actual behavior
  live, since no example exists here to model from.

**After generating, verify the specific column's rendered clause, not just the table's overall
status.** A `calculated_column`'s expression is woven directly into the generated `CREATE TABLE`
statement — a table-level "generation successful" result confirms the statement compiled, not that
any one column's `calculated_field_query` rendered the intended expression/`PERSISTED` syntax. Read
the generated DDL back and check that specific column's clause before considering a newly added
calculated column done.

### Writing an `expression` query

The current row is available as `t1` — every confirmed live example correlates back to it. (The
`concat()` use below is unaffected by the PostgreSQL IMMUTABLE caveat above — `expression` is a
correlated subquery re-evaluated per query, not a stored generated column.)
```sql
-- translation fallback (table has a *_translated companion)
isnull(
  (select t.name from employee_function_translated t
   where t.appl_lang_id = session_context(N'tsf_appl_lang_id')
     and t.employee_function_id = t1.employee_function_id),
  t1.name)

-- composite identity pulled from other tables
concat(
  (select description from project where project_id = t1.project_id),
  ' | ',
  (select name from sub_project where sub_project_id = t1.sub_project_id))
```
Keep the query to exactly the scalar value needed — one narrow subquery per related fact, not a
sprawling multi-join formula — since it re-runs on every row read, not once.

### Performance: `expression` columns get expensive fast

An `expression` column is not indexed, not materialized, and not free: it's a correlated subquery
the generated SQL re-runs for every row returned, every time the table (or anything that shows it,
e.g. as a `look_up_display_col_id` — see "Look-up display column" below) is queried. This is easy to
underestimate because it reads back identically to a real column, with no visible sign of the cost.

**Do**:
- Prefer `calculated_column` over `expression` whenever the logic is same-row only — it costs
  nothing beyond a normal column read, and can be indexed if `PERSISTED`.
- Mark a `calculated_column` `PERSISTED` when it's read far more often than the source columns are
  written, or needs to be filtered/sorted/joined/indexed on — trading a small write-time cost and
  storage for a much cheaper read.
- Keep an `expression` query narrow (one scalar value, minimal joins) and make sure whatever column
  it correlates on, on the *other* table, is indexed — an unindexed correlated subquery is the single
  most common way an `expression` column quietly gets slow as data grows.
- Check the generated SQL/execution plan for any `expression` column used somewhere high-traffic (a
  `look_up_display_col_id`, a default-visible grid column) before shipping it.

**Don't**:
- Don't reach for `expression` when the formula is really same-row logic — that just forces a
  per-row subquery where a free computed column would do.
- Don't stack multiple joins/subqueries into one `expression` when the value could be pulled from a
  single, well-indexed lookup instead.
- Don't use a calculated column (of any kind) as a substitute for a value that could just be set once
  at write time (a Default control procedure, or plain application logic) — recomputing something on
  every single read is wasted work when the inputs rarely change.
- Don't assume `PERSISTED` is always the right call on a `calculated_column` — a virtual
  (non-persisted) one is cheaper to write and perfectly fine for a formula that's read rarely.

## Column order
- **Primary key columns come first**, in the order they appear in the key. For weak entities and link tables, that means the foreign key(s) to the parent(s)/linked table(s) come before the entity's own discriminating column(s) — mirroring the strong → weak ordering rule for composite primary keys.
- **If the primary key includes an identity column, that identity column always comes last within the primary key** — every foreign-key-shaped key column precedes it. A pure link table (rule 3 above) has no identity column at all; a weak entity that needs one (e.g. a historized association, or a classic detail like `sales_order_line`) puts the parent foreign key(s) first and the identity column last.
- After the primary key, place **other foreign keys / reference columns** next, followed by the table's **regular data columns**.
- If a table has **trace/audit columns** (created/modified by + date), put them last, since they're metadata about the row rather than business data — keeps the meaningful columns together and predictable to scan. This is ordering guidance only, not a prompt to add them — see "Integrity, structure, and consistency" below on when (not) to add them.
- **Order-number increments**: use the column's order-number (sequence) property to control this layout, and leave gaps rather than numbering consecutively — **increase by 10 for each column, starting at 10 for the first column** (10, 20, 30, …). This leaves room to insert a column later at, e.g., 15, without having to renumber every column after it.

**API quirk, verified live**: when scripting table creation through a metadata-driven modeling API rather than the UI, a column added before the table's real primary key has been committed can silently default `primary_key = true` (which then also forces `mand`/mandatory to read-only-`true`), regardless of what was requested for that column. This happens with no error — the write reports success. After the intended primary key column is committed, re-read the rest of the table's columns and explicitly correct `primary_key`/`mand` on any that picked up the wrong default rather than assuming the original request held.

## Grid column visibility

**A grid should only show the columns a user is likely to need at a glance** — not every column the
table has. Decide each column's `grid_type_of_col` (`editable`/`read_only`/`hidden`) deliberately
rather than leaving it at its default (which behaves as `editable`, i.e. visible, for every column):

**There is also a third, base-level field, `col.type_of_col`, separate from both
`grid_type_of_col` and `form_type_of_col`** — verified live, and easy to miss since only the
grid/form-specific pair is documented above. In the Software Factory's own column-properties
screen it surfaces simply as "Column type", grouped with general settings like domain/primary
key/mandatory rather than anywhere near the grid- or form-specific settings, which is why it's
easy to change the presentation-specific fields and still leave this one at its default. Same
`editable`/`read_only`/`hidden` enum. Set it together with `grid_type_of_col`/`form_type_of_col`
whenever a column's visibility decision is meant to hold everywhere (not just one presentation
surface) — including the inherited-primary-key carve-out below.

- **Hidden** — surrogate/identity primary keys (meaningless to an end user), audit-only timestamps
  that aren't central to triage (e.g. a "processed on" date when the row's current status already
  conveys that), and secondary/conditional fields that are only populated or relevant for a subset of
  rows (e.g. a rejection reason that's empty except when a row was rejected). These belong on the form,
  not the grid.
- **Read-only** — foreign-key/look-up columns and status-style columns whose value is meant to change
  only through a task/control procedure rather than a direct grid edit (see "Status columns" below for
  the same read-only default applied to the underlying column itself). Still shown, just not
  inline-editable from the grid.
  - **Carve-out**: on a weak entity's own detail tab, the FK column(s) that form the *inherited*
    part of its primary key (e.g. `customer_id` on `customer_address`, per "Column order" above)
    default to **Hidden** instead, not just read-only — its value is already implied by which parent
    row the detail is scoped under, so showing it as a read-only column adds nothing. Only show it
    (read-only or editable) if the user specifically asks for it, e.g. because the table is also
    browsed unscoped, outside its normal parent-filtered context. **Apply this on all three fields —
    `type_of_col`, `grid_type_of_col`, and `form_type_of_col`** — unlike the general "form defaults
    to visible/editable regardless of the grid" independence described below, this specific column
    is redundant everywhere for the same reason, so set all three to hidden together rather than
    only the grid.
- **Editable/visible** (the default) — reserve this for the columns that actually answer "what is this
  row, and what state is it in" at a glance: a handful of core identifying columns, the table's primary
  business figure (an amount, a quantity), and its key status. If a table's column count means most
  columns end up hidden or read-only, that's expected — a wide table rarely needs a wide grid.

This is a deliberate per-table design pass, the same as grouping/sort/search/filter above — decide it
once when the table's columns are created rather than leaving every column at its default and
revisiting later.

**Grid visibility is independent of form visibility — deciding one says nothing about the other.**
`grid_type_of_col` only controls the grid; `form_type_of_col` defaults to visible/editable regardless
of what the grid is set to. The platform's default detail screen still shows a record's form even when
the table's entire grid is read-only or task-driven (e.g. a history table where every write goes
through a task) — so a table designed to have "nothing editable" can still show every column, including
surrogate/identity ones, on the form. Before locking down a table's grid, separately decide (or ask the
user) whether the table's form should be visible at all, and size its column grouping (see "Form & grid
groups" below) against what the *form* actually shows, not against the grid.

## Form & grid groups

Columns can be visually grouped on the form and, independently, in the grid header. Verified live
against the Software Factory's own meta-model (`col`, hundreds of tables): this is a consistently
applied convention, not an occasional nicety, and it should be **planned at the same time as column
order — before any column is created** (see "Decide the group plan up front" below), not bolted on
afterward.

This is about visually banding *columns* under a shared header, on the form and/or the grid. It's a
different mechanism from grouping grid *rows* into a collapsible tree by column value — see "Grid
row grouping and aggregation" below for that.

**Mechanism** — two independent pairs of fields on `col` (the same pattern exists on `task_parmtr`
for task forms, see `thinkwise_software_factory_tasks`):

| Context | "starts a new group" flag | Group label | Section-break variant |
|---|---|---|---|
| Form | `form_field_in_next_grp` (bool) | `form_next_grp_label` (string) | `field_on_next_tab` + `next_tab_label` — pushes onto a whole new tab, not just a new heading |
| Grid header | `grid_field_in_next_grp` (bool) | `grid_next_grp_label` (string) | — (grids have no tab concept) |

The flag and label are set **on the column that starts the new group** — not on the column that ends
the previous one. Form and grid grouping are independent: a grid can group columns differently than
the form, though in practice most tables just group the form and leave the grid ungrouped.

**Switching a column between the group and section-break variant needs two writes, not one.** The
two mechanisms are mutually exclusive on a given column, and turning `form_field_in_next_grp` off
also flips `form_next_grp_label` to a non-editable/hidden state — if the same write also tries to
clear that label (e.g. set it to null) or set `field_on_next_tab`/`next_tab_label` in one combined
call, the label write can be rejected because the field became non-editable partway through applying
that same call. Clear the old mechanism's flag first (as its own write), then set the new mechanism's
flag and label in a second write, rather than attempting the swap in one call.

**When to use them**: any table beyond a handful of columns, or with visually distinct concerns
(identity vs. status vs. settings vs. description) — group them. Don't group a table with only 2-3
closely related columns; there's nothing for a heading to separate.

**Which columns to bundle together**:
- First group holds the identifying/core descriptive columns (conventionally labeled `general`).
- One group per cohesive concern after that (`status`, `settings`, `description`, `positioning`,
  `authentication`, …) — never fold unrelated concerns into one group just to reduce group count.
- If the table has trace/audit columns (only when the user actually asked for them — see "Don't add
  trace/audit columns on your own initiative" under "Integrity, structure, and consistency"), the
  standing convention is a trailing group labeled `mutation`, additionally flagged
  `field_on_next_tab=true, next_tab_label="trace"` so the audit columns live on their own tab instead
  of cluttering the main form — this holds regardless of the table's subject matter.

**Naming**: lowercase snake_case, 1-3 words, a topic noun (`status`, not `"Status info"` or
`"status_columns"`). **Reuse an existing label instead of inventing a new one whenever the meaning
matches** — the real model reuses a small vocabulary of maybe 20-30 labels across hundreds of tables
(`general`, `description`, `status`, `settings`, `progress`, `positioning`, `assignment`, `tag`,
`generation`, `mutation`, `authentication`, `user_interface`, `condition`, `query`, `filter`,
`default_value`, …). This isn't just tidiness — see Translation below for why it's the whole point.

**Translation**: a group label's text is its own `transl_object_transl` row, keyed by the **literal
label string**, not by table+column — confirmed live: the label `general` has exactly one translation
row per language, shared and reused by every table that uses that label, already approved in ~17
languages. Reusing an existing label costs zero additional translation work. Inventing a new
spelling/casing variant (`"Status"` vs `"status"` vs `"state"`) creates a brand-new untranslated
object that has to go through the whole translation/approval cycle for no modeling benefit. After
introducing a genuinely new label, follow `thinkwise_software_factory_translation_objects` to fill
in and approve it — the same standing requirement as any other new translatable object (see
"Translating new objects" above).

### Decide the group plan up front, before creating any columns

Work out the full grouping (which columns belong to which group, what each group is labeled, and
where any tab-breaks fall) as part of the same design pass as column order (see "Column order"
above) — **before** issuing the first `stage_resource`/create call for the table's columns, not as a
follow-up editing pass once the columns already exist. Concretely: when creating column N, its
`order_no`, `form_field_in_next_grp`, `form_next_grp_label` (and `field_on_next_tab`/`next_tab_label`
if it starts a new tab) should all be set **in the same write** that creates the column. Planning
this upfront and setting it immediately avoids a second round of patch calls per column purely to add
grouping after the fact — each column is touched once instead of twice.

**The plan must cover every column, including the first one.** A leading identity/PK column is easy
to treat as exempt from grouping because it precedes the first semantic group, but it still needs a
group of its own (e.g. an "Identity" group over the record's own id and any leading display-name
field) — a Form where the first column alone sits outside every group is still an incomplete grouping
plan, not a finished one.

### Keep the group plan current when the table changes later

The "decide up front" rule above covers a brand-new table's first Form. The same discipline applies
afterward:

- **Adding a Form to a table for the first time, after the table already has columns** — treat it
  exactly like new-table design: decide the full group/section plan in one pass before setting any
  group flags, not incrementally per column.
- **Adding a single new column to a table whose Form already has groups/sections** — don't append
  the new column ungrouped at the end by default. Match it to whichever existing group covers the
  same concept, setting its order/group flags in the same write that creates the column. If no
  existing group is an obvious semantic fit, ask the user which group it belongs in (or whether it
  needs a new one) — see `thinkwise_software_factory_mcp_base`'s "Ask, don't default" rule — rather
  than guessing.
- **Retrofitting groups onto a table that already has ungrouped columns** — check the *first* visible
  column too, not just the ones after the first existing group starts. It's easy to group everything
  from the first semantic boundary onward and leave the leading identity/PK column(s) stranded outside
  any group simply because nothing preceded them to trigger the check.

## Sort, search, and filter

Every column has independent per-column settings for the table's default sort, its participation in
find/search, and its participation in the filter panel — verified live on `col`:
`default_sort`/`sort_no`/`sort_order`/`allow_sort` (sort), `visible_for_search`/`search_order_no`/
`search_condition`/`include_in_global_filter` (search/find), and `visible_for_filter`/
`filter_order_no`/`filter_condition` (filter). `visible_for_search` and `visible_for_filter` share
the same three-way enum: `always` / `extended` / `never`.

### Sort

**Every table should have a default sort set on the most sensible column(s)** — e.g. a natural
ordering column (`order_no`), a name/code, or a date (often descending, for "most recent first").
Set `default_sort=true` on each column that participates, `sort_no` to control precedence when more
than one column is involved (a composite sort), and `sort_order` (`asc`/`desc`) per column. Leave
`allow_sort=true` (the default) on any column a user could reasonably want to sort by; only turn it
off for columns where sorting is meaningless (large text/blob columns — see below).

**When the sensible default sort isn't obvious, ask the user rather than guessing** — unlike filter
and search below, this isn't a "safe default, override later" setting: a wrong default sort is
visible on every screen open and is a judgment call about the business data, not a mechanical rule.

### Search (find)

**A subject (strong entity, or the parent side of an inheritance relationship — see "Tables" above)
should always have search configured if its screen type has a grid.** Don't leave `visible_for_search`
at its unconfigured default (which behaves as `always` on every column) or skip search setup entirely
— deliberately choose `always`/`extended`/`never` per column following the guidance below, the same
required-follow-up treatment as translations and menu placement get elsewhere in this skill.

**Restrict `visible_for_search=always` to the table's genuinely important columns** — the ones a
user would actually type into a quick-find box to locate a row (names, codes, key statuses). Set
`visible_for_search=never` on any column backed by a large-object type — `NVARCHAR(MAX)`/
`VARCHAR(MAX)`, `VARBINARY(MAX)`/`IMAGE`, `TEXT`/`NTEXT`, `XML` (see "Data type recommendations"
above) — searching these is either meaningless (binary) or expensive (unbounded text) and never
what a quick-find is for. Default every other column to `extended` (available under advanced
find/search, not cluttering the default quick-search) rather than `always` — mirroring the filter
default below. Set `search_order_no` for a sensible position among the columns that do participate,
and `search_condition` to the operator that makes sense for the column's data (`contains` for free
text, `equal_to` for codes/numbers/domain-element-backed columns).

**`visible_for_search`/`search_condition`/`search_order_no` alone don't make a column searchable —
`include_in_global_filter` (a separate boolean, "Include in search") is the flag that actually puts
the column into the searchable set.** Set `include_in_global_filter=true` on every column that gets
`always` or `extended`, alongside its `visible_for_search`/`search_condition`/`search_order_no` —
missing this step leaves the column configured but silently excluded from search, with no error to
flag it.

### Filter

**Default every column's `visible_for_filter` to `extended`.** Only promote a column to `always`
when it's genuinely one of the most important columns on the screen *and* one of the filters a user
is actually likely to reach for often — treat `always` as the exception that has to earn its place,
not the default. As with search, large-object-typed columns (`NVARCHAR(MAX)`/`VARCHAR(MAX)`,
`VARBINARY(MAX)`/`IMAGE`, `TEXT`/`NTEXT`, `XML`) should be `never` rather than `extended` — they
can't be meaningfully filtered at all. Set `filter_order_no` to position the `always`/`extended`
columns sensibly, and `filter_condition` to the operator that fits the column (`contains` for free
text, `equal_to` for codes/domain-element-backed columns, `between` for ranges/dates).

**For the design reasoning behind these settings — which sort pattern fits which subject type, which
columns actually belong in search vs. filter, the deep-join/filter-form caution, tables-vs-views for
presentation reasons, and lookup-subject design — see
`references/subject_presentation_design.md`.** This section covers the fields; that file covers why
and which.

### Decide all three before creating any columns

Exactly like grouping (see "Form & grid groups" above), work out sort/search/filter settings for
every column as part of the same upfront design pass as column order and grouping — **before** the
first `stage_resource`/create call for the table's columns. When creating column N, its `order_no`,
group fields, `default_sort`/`sort_no`/`sort_order` (if applicable), `visible_for_search`/
`search_order_no`/`search_condition`, and `visible_for_filter`/`filter_order_no`/`filter_condition`
should all be set **in that same write**. The only exception is a genuinely uncertain default-sort
column — confirm that one detail with the user, then include it in the same create call once
confirmed, rather than leaving every column's settings for a separate patch pass afterward.

## Grid row grouping and aggregation

Distinct from the grid *column* header grouping in "Form & grid groups" above
(`grid_field_in_next_grp`/`grid_next_grp_label`, which visually bands grid *columns* under a shared
header) — this covers grouping grid *rows* into a collapsible tree by one or more column values,
plus per-column footer totals. Verified live against real application models (`INSIGHTS`/
`INSIGHTS_DEMO`, a time-tracking/invoicing app) as well as the Software Factory's own meta-model.

**Mechanism**:
- Table-level (`tab`): `allow_grp` (bool) turns the feature on for the table's grid at all;
  `grp_box_visibility` (`never`/`when_grouped`/`always`) controls whether the drag-to-group drop
  area is shown to end users; `grp_grid_default_expanded` (bool) + `grp_grid_default_expanded_level`
  (byte) control whether the default grouping starts expanded, and how many levels deep.
- Column-level (`col`): `grp_until` (bool) — flip it on the column(s), in grid display order, that
  should form the default group-by hierarchy. Every column from the first up through the last one
  flagged `grp_until=true` becomes a grouping level, in that order (one flag = a single-level default
  group-by on that column; flag more than one, in order, for a multi-level nested group).
- Aggregation (`col`): `show_aggregation_in_grid` (bool) + `aggregation_summary_type` (enum: `sum`,
  `count`, `average`, `min`, `max`, `stddev`, `stddevp`, `var`, `varp`) puts a per-column summary in
  the grid's footer — and in each group's own footer row too, when grouping is active on the same
  grid.

### When to use grouping

Real usage clusters into two shapes:
- **Worklist/overview/junction-style grids** — many flat rows that are more scannable collapsed under
  a categorical owner/type dimension. Confirmed live: `validation_msg_assignment_overview` groups by
  `assigned_to_developer_id` (validation issues collapse per developer); `deployment_module_role_overview`
  groups by `role_grp_id` (permissions collapse per role group). `grp_box_visibility` is `never` on
  nearly every one of these in the reference model — the grouping is a fixed, developer-chosen
  default the user isn't expected to rearrange, not an open-ended ad hoc feature.
- **Genuinely ad hoc, user-driven grouping** — rarer; set `grp_box_visibility` to `when_grouped` or
  `always` only when end users should be able to drag arbitrary columns into their own grouping, not
  just view a fixed default.

Detail/transactional grids reached from a parent's tab (e.g. `hour`, `booking_hour`, `sub_project` in
the reference model) typically leave `allow_grp=false` entirely — they're already scoped to one
parent row, so an in-grid group-by adds nothing.

**When it's not obvious whether a table's grid should default-group by something, ask the user**
rather than picking a column — like default sort, this is a judgment call about how the business
data is actually consumed, not a mechanical rule.

### When to use aggregation

Two distinct, real patterns, independent of whether grouping is also used:
- **`sum` on genuine numeric measure columns** — money amounts, hours, quantities, durations (e.g.
  `amount_incl_vat`, `number_of_hours`, `hours_booked`, `function_points`). Gives a subtotal per
  group and a grand total in the grid footer. Used with or without grouping — `hour`/`booking_hour`/
  `sub_project` show grand-total sums with no grouping at all, since they're already scoped under one
  parent.
- **`count` on any already-visible, always-populated column** (very often the primary key, or a
  status/icon column already on the grid) purely to show a row count per group and overall —
  confirmed live on columns like `col_id`, `test_scenario_id`, `unit_test_id`, `change_log_status`,
  `icon`. This is the cheap way to get an "N records" indicator without adding a dedicated column
  just to count rows — the column chosen for `count` doesn't need to be meaningful in itself.
- `average`/`min`/`max`/`stddev`/`stddevp`/`var`/`varp` exist but are rare in practice — reserve them
  for grids genuinely doing statistical/range analysis, not typical business data.

**When it's unclear whether a numeric column should be summed (or which column should carry a
`count`), ask the user** — same reasoning as grouping and default sort.

### Decide grouping and aggregation before creating any columns

Same rule as grouping, sort, search, and filter above: work out which column(s) (if any) get
`grp_until`, and which column(s) (if any) get `show_aggregation_in_grid`/`aggregation_summary_type`,
as part of the same upfront design pass — before the first create call for the table's columns — and
set the table-wide flags (`allow_grp`, `grp_box_visibility`, `grp_grid_default_expanded[_level]`) at
the same time the table itself is created. Set each column's `grp_until`/aggregation fields in the
same write that creates the column, avoiding a second patch pass once the columns already exist. The
only things worth pausing for user confirmation are which column(s) should drive the default
grouping and which numeric column(s) should be summed — exactly as with default sort.

## References (foreign keys)

**Model the reference as part of designing the table (or view) — not as an afterthought added once screens are already being built.** Whenever a column's value is meant to match another table's primary key, that relationship must be represented by an actual `ref` row (plus one `ref_col` row per join column). Two columns that merely happen to hold matching values are not a reference to the platform: without a modeled `ref`, there is no look-up combo, no detail grid, no auto-derived OData/Indicium navigation property, and (when `check_ref` is on) no database-level integrity check. Add the `ref`/`ref_col` at the same time you add the FK-shaped column, before moving on to screens, tasks, or reports that would want to use it.

### Look-up vs. detail: what one reference gives you

A single reference always has two possible presentations, toggled independently via `show_look_up` / `show_detail` on the `ref` row:
- **Look-up** — a field/combo on one table's screen letting a user search and select a single related row on the other table. This belongs on the table that actually holds the FK-shaped value — it is picking "the one" row it relates to.
- **Detail** — a grid/tab on one table's screen listing every row on the other table whose key matches. This belongs on the table being pointed at — it has "many" related rows to show.

**Direction — one unified rule, verified live, no exceptions:** `source_tab_id` = the table whose primary key is being referenced (the "parent"); `target_tab_id` = the table *or view* holding the FK-shaped column that points at it (the "child"). This is identical whether the child is a normal table or a view — there is no separate "view case" that reverses anything; a view with an FK-shaped column follows exactly the same direction as any other child table. Mechanically, `ref_col.source_col_id` must be a column that is part of `source_tab_id`'s own primary key, while `ref_col.target_col_id` is simply the FK-shaped column on `target_tab_id` holding the matching value — that's the one fact to check if direction is ever in doubt (see verification note below). Look-up goes on the target (the FK-holder, since it's picking "the one" parent row it relates to); detail goes on the source (the PK-owner, since it has "many" children pointing at it).

Example — `absence.employee_id` → `employee.employee_id` (a normal table-to-table FK):
```
ref:      source_tab_id = employee,   target_tab_id = absence
ref_col:  source_col_id = employee_id,   target_col_id = employee_id
```
Result: `absence`'s screen shows a look-up to pick the employee; `employee`'s screen shows a detail grid of that employee's absences.

Example — a view `employee_absence_overview` with a column `employee_id` that should look up `employee`:
```
ref:      source_tab_id = employee,   target_tab_id = employee_absence_overview,   check_ref = false,   show_detail = false
ref_col:  source_col_id = employee_id,   target_col_id = employee_id
```
Same rule, same direction as the table-to-table example above — `check_ref = false` here is because a view carries no physical FK constraint (not because the direction itself changes), and `show_detail = false` is typical since a detail grid on the real table listing view rows is rarely useful.

Confirmed against real production references in a live model (`ref_company_dev_event_company`, `ref_person_dev_event_company_account_manager`, both against a view target).

**When a table ends up with more than one detail tab** (multiple references showing `show_detail =
true` against the same parent), give each a distinct order number — two detail tabs left at the same
position is flagged by a Software Factory validation and produces an ambiguous tab order in the UI.

**Verification note, and why this matters more than it looks:** an earlier version of this document got the table-to-table case exactly backwards (`source` = FK-holder, `target` = PK-owner) — a plausible-sounding assumption from generic ER-modeling habit that was never actually checked against a live model, sitting right next to a view-case rule that *had* been verified and was correct. The wrong version committed without any write-time error and wasn't caught until a `check_ref = true` reference tripped a live validation rule resembling "foreign key reference with integrity does not contain the full primary key of the source table." **When a reference's direction is in doubt, don't rely on documentation or intuition alone — query the platform's own live validation/diagnostics for the model** (most Software Factory-style connectors expose one) after modeling it. A rule flagging that the FK-shaped column isn't part of the "source" table's primary key means source/target are swapped, full stop — this is faster and more reliable than re-deriving the rule from memory every time.

**Disambiguating multiple references to the same table pair:** if a table (or view) has two or more FK-shaped columns pointing at the same target, the auto-derived `ref_id` (`ref_<source>_<target>`) collides between them. Set `ref_add` on each (e.g. `"most_recent"` / `"upcoming"`) — it appends a suffix (`ref_<source>_<target>_<ref_add>`) so each gets a distinct primary key instead of silently overwriting the first.

**Self-referencing foreign keys can't cascade on SQL Server, verified live.** A reference where `source_tab_id` equals `target_tab_id` (a table's own parent-child hierarchy, e.g. a `parent_id` pointing back at the same table) fails to deploy with `on_delete`/`on_update` set to anything other than `no_action` — SQL Server rejects the constraint outright at deploy time ("Introducing FOREIGN KEY constraint ... may cause cycles or multiple cascade paths"), even though the identical `on_delete = set_null`/`cascade` value deploys fine on an ordinary two-table reference. Model a self-referencing FK with `on_delete = no_action` and `on_update = no_action`, and if reparenting/promote-to-top-level-on-delete behavior is actually wanted, implement it in a control procedure or at the application level rather than relying on the database constraint to do it. Verify the same restriction independently before assuming it holds on a non-SQL-Server target dialect.

## Look-up display column

Every table should have a deliberately chosen `tab.look_up_display_col_id` — the column (or
calculated column) used to represent one of its rows wherever the table is looked up from elsewhere:
combo/auto-complete fields, look-up popups, detail headers. Leaving it unset or pointing at the
wrong column means users see a raw ID or a meaningless field the moment the table is referenced from
another screen. (`tab.tree_display_col_id` is the equivalent for tree views, when used.)

Two patterns, both confirmed live against a real application (`INSIGHTS`):

- **Point it at an existing descriptive column**, when one already identifies the row well:
  `customer` → `name`, `employee` → `name`, `project` → `description`, `task`/`meeting`/`email` →
  `subject`, `sales_invoice` → `sales_invoice_description`.
- **Add a dedicated calculated column** (see "Calculated columns" above) when no single column does
  the job:
  - **Translation fallback**, for a table with a `_translated` companion table — resolve the row's
    name in the session's current language, falling back to the base-language column. Confirmed
    live on `activity`, `country`, `document_type`, `employee_function`, `meeting_type`.
  - **Composite/concatenation**, when the row's identity is really a combination of several related
    fields — e.g. a real live model has `hour.lookup` = `project.description + ' | ' + sub_project.name`;
    `booking_hour.lookup` = employee name + the ISO week number; `declaration.lookup` = project +
    employee + description + date, joined together.

**Never name this column the generic `lookup`, despite that older live examples above use exactly
that** — it's the same meta-information-free naming this document rules out everywhere else (see
"General naming rules"). Give it a name that says what it actually computes: `full_name` for a
first+last name concatenation, `full_address` for an address composite, `display_label` for a
translation fallback, and so on. A calculated column built this way is almost always an
`expression`-type calculated column, not a same-row `calculated_column` — it exists specifically to
reach into other tables. Keep it narrow (see "Calculated columns" → Performance above) since it runs
once per visible row everywhere the table is looked up from, which can be a lot of places.

**The chosen display column (or calculated column) should return a distinct value per row.** A
look-up display column that returns duplicate values across rows is flagged by a Software Factory
validation, since it leaves a user unable to tell two rows apart in a combo/auto-complete. If no
single existing column is reliably unique, prefer the composite/concatenation calculated-column
pattern above over accepting a display column with likely duplicates.

**Decide this at the same time as the rest of the table's design** — before the column-creation
pass, the same as grouping/sort/search/filter/aggregation above: know whether an existing column
will serve, or whether a dedicated calculated column is needed, what it should compute, and what to
name it, so
`look_up_display_col_id` (and the calculated column itself, if needed) get set in the same
create-time writes instead of a follow-up patch pass.

## Diagrams (one per functional/subject area)

**"Domain" here means a functional/subject area** — an informal business grouping like "Sales", "HR",
or "Finance" — not a Software Factory object in its own right (don't confuse it with the DTTP `dom`
entity in the "Domains" section below, which is a data type, not a diagram scope). Software Factory has
no dedicated entity for this kind of grouping — a **diagram** (`diagram`, plus `diagram_tab`/`diagram_ref`
placement rows) *is* how a functional/subject area gets represented in the model, in principle one
diagram per area.

### Currently not possible via this kind of API — don't attempt it

**Placing a table (or a reference) onto a diagram is currently blocked on every write path tried,
verified live**, both on a metadata-driven staging API and on a direct entity write:
- `task_create_own_diagram` is rejected before any field can even be set.
- A direct add to `diagram_tab` (the join row that positions a table on the canvas) is rejected even
  after supplying the required parent context.
- `task_add_table_to_diagram` itself takes **zero parameters** — there is no way to target a specific
  table with it even in principle.

**Don't create a new diagram, and don't try to add a table or reference to an existing one.** Creating
the bare `diagram` record itself (a plain add, with nothing placed on it) does commit fine, but an
empty diagram isn't useful, so there's no reason to create one either while this is blocked. Treat all
diagram/table/reference placement — including via the other bound tasks listed below
(`task_one_level_deeper`/`task_one_level_higher`/`task_create_link_table`/`task_copy_diagram`/
`task_import_into_own_diagram`) — as a manual step in the Software Factory's own UI. If a request would
otherwise call for adding to or creating a diagram, say so explicitly and describe the manual step for
the user, rather than attempting the API call and only falling back once it fails.

The subsections below still describe the *design* decision (which diagram a table or reference should
logically live on, when a diagram has outgrown itself) — that reasoning is still worth handing to the
user as part of the manual step. Nothing in them should be executed as an actual write against
`diagram`/`diagram_tab`/`diagram_ref` until this limitation is lifted.

### Adding a new table (subject): which diagram does it belong on?
- Identify which functional/subject area the new table belongs to.
- **If a diagram already exists for that area**, tell the user the table should be added to it as a
  manual step (see above) — don't call `task_add_table_to_diagram`.
- **Only recommend a new diagram** — for the user to create manually via `task_create_own_diagram`
  then `task_rename_diagram` in the Software Factory UI — **when the table starts a genuinely new
  functional/subject area** not represented by any existing diagram yet.
- **Still flag every newly created table's diagram placement as an open item**, even though it can't be
  done through the API right now — an undiagrammed table is invisible to anyone reviewing the model
  visually, so don't let this limitation become a silent reason nothing ever gets diagrammed.

### Adding a new reference: which diagram does it belong on?
- **If both tables of the reference already appear on the same area's diagram**, tell the user the
  reference should be added there too (manually) so the relationship is visible where the tables
  already are.
- **If the reference crosses two different areas' diagrams** (e.g. `sales_order.employee_id` →
  `employee`, where `employee` lives on the "HR" diagram and `sales_order` on "Sales"), recommend
  placing the reference on the diagram of the table that conceptually owns the relationship (usually
  the referencing/child table's area) and pulling in just that one related table, rather than merging
  two areas into one diagram — again, as a manual step, not an API call.

### When a diagram has genuinely outgrown itself
- A diagram that's grown past the point of being readable at a glance (dozens of tables, dense crossing
  reference lines) is a signal to split it — there's no fixed table-count threshold; judge it by whether
  it's still doing its job as an at-a-glance picture of one functional/subject area.
- Splitting is not "shrink every diagram to some size" — keep the split aligned to real functional
  boundaries. If a diagram is oversized because its area genuinely covers more than one concern, split
  along those functional lines (e.g. "Sales" into "sales_ordering" and "sales_reporting") rather than an
  arbitrary table-count cut. This is design guidance to hand to the user; the actual split/copy work
  (`task_copy_diagram`/`task_import_into_own_diagram`) is a manual step for the same reason as above.

### Bound tasks on `diagram` — for reference only, not currently usable through this kind of API
- `task_add_table_to_diagram` — place one table onto a diagram.
- `task_one_level_deeper` (given a `tab_id` already on the diagram) — pulls in every table one hop away
  via a reference to/from it.
- `task_one_level_higher` — the reverse; trims tables one hop out, for narrowing a diagram back down.
- `task_create_link_table` — models a many-to-many link table directly from the diagram, wiring it to
  two tables already placed on it.
- `task_copy_diagram` / `task_import_into_own_diagram` — duplicate or merge an existing diagram's
  layout rather than starting a split or overlapping diagram from scratch.

These still exist in the metamodel and may work directly in the Software Factory's own UI — only the
API write path is confirmed blocked (see above). Re-verify before relying on this list again if the
platform/connector version changes.

## Domains
- **No data type (DTTP) in the name** (exceptions: `XML`, `DATE`, `IMAGE`, where the type is the concept itself).
- **No length specification in the name** — a domain named `code_10` becomes wrong the moment the length changes.
- **No other meta-information** — describe the *business concept* (`email_address`, `currency_code`) so the domain can be reused consistently, not its implementation.
- Design for reuse: if two columns represent the same kind of value, they should share a domain rather than each defining an inline type.
- **Never create/use a bare `id` domain for a primary/foreign key.** A generic `id` domain is exactly the kind of meta-information/no-context name this section rules out elsewhere — it doesn't say what it identifies, and it also invites collisions/reuse across unrelated keys that happen to share a data type. Name the domain after the specific entity it identifies, matching the column name it backs: domain `employee_id` for `employee.employee_id`, domain `sales_order_line_id` for `sales_order_line.sales_order_line_id` — not a shared `id` domain. Wrong: domain `id` used for both `employee.employee_id` and `sales_order_line.sales_order_line_id`. Right: separate `employee_id` and `sales_order_line_id` domains, one per entity. **Exception**: if the model being expanded already shares one generic `id`-style domain across essentially every table's surrogate key, see "When a guideline conflicts with an existing model's own established convention" above before introducing a new per-entity convention only for the objects you're adding.
- **This exact-match convention breaks on PostgreSQL, verified live via the Software Factory's own generation validation**: "PostgreSQL cannot have the same name for different objects within the same schema" — a domain and the column it backs cannot share an identical identifier on a Postgres branch, which the naming rule above produces by design (domain `employee_id` backing column `employee.employee_id` is exactly a same-name collision). This isn't limited to ID domains — any domain deliberately named identically to its column collides the same way (a live case: domains `address_type`/`address_line`/`postal_code` matching columns of the same name).
  **Check the branch's `rdbms_type` (see "Data type recommendations" below) before creating any domain. On a PostgreSQL branch, prefix every domain you create with `dom_`** (`employee_id` → `dom_employee_id`, and just as much for a non-ID domain like `email_address` → `dom_email_address`, even though that one wouldn't collide) — a blanket prefix applied to every domain, not a case-by-case check for whether *this* domain's name happens to match a column. Checking column-by-column doesn't scale (it's easy to catch it for one obvious ID domain and still miss it for an ordinary descriptive one) and produces a model where some domains are prefixed and others aren't for reasons no one remembers later. If expanding a model that already has a pre-existing, unprefixed domain, rename it too for consistency (via the dedicated rename task below) rather than leaving the model half-migrated.
  Use the dedicated rename task, not delete-and-recreate, and re-run the branch-wide placeholder-translation query afterward as usual — domains themselves don't carry their own `transl_object`, so this is normally a no-op, but confirm rather than assume.
- **Field-naming trap when querying/scripting domains via metadata**: the entity itself is `dom`, and a
  column's foreign key to it is `dom_id` — not `domain_id`, despite "domain" being the natural English
  word for the concept. There is also no single `type_of_domain`-style field on `dom` — its data type
  lives in `dttp_id`/`dttp`, and its UI control in `control_id`. Confirm exact field names via the
  entity's own live metadata before guessing either one; both wrong guesses fail with the same
  "column could not be found"-style error.
- **Same naming trap for domain elements**: the rows themselves live on an entity set named
  `elemnt`, not `dom_elemnt` — even though `dom_elemnt` exists as its own distinct object-type
  identifier elsewhere in the metadata, which makes it a plausible but wrong guess. Confirm via
  `dom`'s own navigation properties (the one pointing at domain elements targets entity set
  `elemnt`) rather than assuming the name that "reads" correctly.

**API quirk, verified live**: a column's `mand` (mandatory) flag can revert to its domain's own default `mand` after the domain is assigned, even if the column's own `mand` override was set in the same or an immediately following write. This showed up on view columns that needed to be nullable (e.g. an optional end date) despite their domain defaulting to mandatory. Don't assume a combined "assign domain + set mand" write holds — after assigning a domain to a column that needs a different mandatory setting than the domain's default, re-read the column back and re-apply `mand` if it didn't take.

## Data type recommendations

**These recommendations assume a SQL Server target — check the branch's actual RDBMS before applying
them.** A branch's RDBMS is set on `branch_rdbms_type`; query `dttp` filtered by that same
`rdbms_type` to see the actual set of types available on this branch before picking one — don't
assume a type mentioned below exists. **Verified live on a PostgreSQL branch: there is no
`NVARCHAR` and no `TINYINT` at all** — use `VARCHAR` (PostgreSQL has no fixed/variable Unicode
distinction, so plain `VARCHAR` is the sensible choice) and `SMALLINT` (the smallest available
integer type there) respectively wherever the guidance below says `NVARCHAR`/`TINYINT`. Other
non-SQL-Server platforms (Oracle, iSeries) likely have their own equivalent substitutions — verify
the same way rather than assuming.

- `DATETIME2` instead of `DATETIME`.
- `NVARCHAR` instead of `VARCHAR`, unless the character set must be restricted, in which case `VARCHAR` is acceptable.
- `NUMERIC` for decimals (avoid `FLOAT` due to rounding/precision issues).
- `INT`/`BIGINT` for identity columns.
- For booleans, use `BIT` (`1`/`0`) as the default choice; only use `CHAR`/`VARCHAR` (`'Y'`/`'N'`, `'T'`/`'F'`) if the business genuinely needs textual values.
- Avoid `CHAR`/`NCHAR` unless there's a specific justification (fixed-width values only).

## Domain elements
A **domain** is an abstract data type (Data > Domains) that standardizes the data type, constraints, and default UI control for every column/parameter using it. **Domain elements** are a fixed, pre-defined set of selectable values attached to a domain, used for `COMBO`, `IMAGE COMBO`, and `RADIO BUTTON` controls (e.g. a `payment_method` domain with elements `paypal`, `creditcard`, `prepaid`, `afterpay`).

**When to use domain elements vs. a lookup table** — straight from Thinkwise's SQL coding guidelines: **"Use a domain with domain elements if you need to program on values."**
- Use domain elements when control procedures / business logic need to branch, compare, or react to specific fixed values (e.g. `status = 'approved'`) — elements give each value a stable, named ID to reference in code instead of a magic string/number, with translation handled by the platform.
- Use a lookup/reference table instead when the value set is user-maintainable, expected to grow, needs additional attributes per value, or nobody writes code against specific values.
- Elements support being marked **inactive** (since 2021.2, per-variant since 2021.3) so old values keep working for historic data while hidden from new selections — a workaround for evolving a fixed set, not a substitute for a table when the set is inherently dynamic.

**Setting up and naming domain elements** (Data > Domains > Form):
- **Database value** — unless there's a specific reason to deviate (matching an external system's codes, or a value that must stay stable against an existing integration), use a **sequential integer starting at 0 or 1, incrementing by 1 per element**. Don't make the database value itself descriptive — that's what the ID is for.
- **ID** — the translatable label key shown to users; put all descriptive meaning here (e.g. `paypal`, `creditcard`, `prepaid`), decoupled from storage.
- **Sequence number** — controls display order in the combo/radio list independent of the database value.
- **Availability/active flag** — whether the element can still be newly selected.
- **Data type**: since the database value defaults to a small sequential integer, the preferred data type for a domain that uses elements is **TINYINT** rather than `INT`/`BIGINT` — no need to reserve more range than a handful of fixed options will ever use. Step up only if the database value deliberately isn't a small sequential integer (e.g. must match an external code exceeding 255, or is a bitmask).
- Grid vs. form can't natively show different translations for the same element (the workaround is a duplicated domain + expression field) — keep element ID translations reasonably concise so they work acceptably in both contexts.
- **When the domain's control is `IMAGE COMBO` or an icon-based `RADIO BUTTON`**, each element needs its own `elemnt.icon_id` — that's what actually renders instead of text. Follow `thinkwise_software_factory_icons`'s status-vocabulary guidance (unique silhouette per state, never color/icon alone) and its rule to ask the user rather than guess when no existing icon fits.
- **Setting `dom.control_id` to `IMAGE COMBO` (or another icon-capable control) silently resets `dom.alignment` to `right` for a numeric-typed domain** — even though a left-aligned icon/enum-style presentation is what an element-backed TINYINT domain normally wants. Confirmed live: 4 numeric domains all flipped to `alignment=1` right after their `control_id` was set, while sibling element-bearing domains that hadn't had `control_id` touched stayed at `alignment=0` left. Re-read `dom.alignment` immediately after any `control_id` write on a numeric domain and patch it back to `left` (pass the numeric value `0` — the string key `"left"` is rejected with `invalid_input`) unless the model's own convention is actually right-aligned for that domain type.

## Status columns

**Unless the user states otherwise, a status column defaults to read-only, mandatory, with a default
value:**
- **Mandatory (`mand = true`)** — a row should never sit in a null/undefined status.
- **A default value set** (the column's own default, or a Default control procedure) so every new row
  gets a sensible initial status (e.g. `new`/`draft`) without the caller having to supply one.
- **Read-only on the form/grid** — don't let an end user free-edit the column directly; a status is a
  business state, not a plain data field.

**Unless the user states otherwise, status changes are made by a task or control procedure, not by a
direct column edit.** Model a dedicated task (or one per meaningful transition, e.g. `submit`/
`approve`/`reject`) that performs the update — see `thinkwise_software_factory_tasks` — so each
transition can validate preconditions and trigger side effects, rather than exposing the raw column
for arbitrary overwrite. A status backed by domain elements (see "Domain elements" above) is what
lets that task/control-procedure logic branch on a stable element ID instead of a magic value.

## Conditional layout — consider it, don't default to it

After designing a table's columns (and especially after adding a status column, per above), take one
deliberate pass asking whether **conditional layout** — visual highlighting of a value or row based on
a condition, e.g. status colours, overdue-date highlighting, missing-data warnings, threshold breaches —
would genuinely help a user of this table. The usual candidates: a status/domain-element column, a date
column compared to today or another date, a numeric column with a meaningful threshold, or a column
that's only populated/relevant in an exceptional case.

**Only add one where there's a good candidate — don't add one just because a table has a status
column.** Plenty of tables have no good candidate at all (a pure link table, a table whose state is
already obvious from a domain icon, a settings-only table). When that's the case, say so and add
nothing rather than inventing a marginal layout just to have "covered" the step.

**Never add a conditional layout without checking with the user first.** Present the candidate(s) you
found — which column or row, what condition, what it would communicate — and get explicit confirmation
before creating anything. If this table design is part of a larger plan (e.g. one produced by
`thinkwise_software_factory_build_planner`), fold the candidates into that plan and get the **plan**
confirmed before finalizing it, rather than adding them as a silent addendum once modeling starts.

For the actual mechanics — entity/field reference, the condition enum, row-vs-column targeting,
light/dark colours, and known gaps/pitfalls — see `thinkwise_software_factory_conditional_layouts`.
This section only decides *whether* one is warranted; that skill covers *how* to build it.

## Translating new objects

Every table (`tab`), column (`col`), and domain element (`dom_elemnt`) created following this
skill gets a translation object auto-generated with placeholder text — literally the object's own
ID wrapped in brackets (`[sales_order_line]`, `[customer_code]`) — which is what actually renders
in the running application until it's translated. Creating the object is not the last step for
anything user-facing: after modeling a new table/column/domain element (or a batch of them), follow
the `thinkwise_software_factory_translation_objects` skill to find and fill in these
placeholder-marked objects, the same way "Exposing new tables on a menu" above is a required
follow-up, not an optional polish pass.

**Don't rely on remembering to do this as you go — verify it with a query as the last step of the
task, every time.** A real session translated the new tables it created and still left every one of
their columns sitting at bracket-placeholder text, because "translate the new objects" was followed
as a narrative reminder during the build rather than checked mechanically at the end. Before
declaring any modeling task complete, run this — scanning the **whole branch**, not just the objects
touched this session, since a narrower scan misses pre-existing gaps the task happened to touch in
passing:

```
/transl_object_transl?$filter=model_id eq '<model>' and branch_id eq '<branch>' and startswith(transl,'[')&$select=type_of_object,transl_object_id,transl
```

Zero rows is the actual definition of "done," not "I translated the things I remember creating."
Every skill that creates translatable objects (`thinkwise_software_factory_create_view`,
`thinkwise_software_factory_tasks`, `thinkwise_software_factory_menu`, and any plan produced by
`thinkwise_software_factory_build_planner`) points back to this query as its own final gate rather
than repeating it — treat this section as the canonical source for it.

## Integrity, structure, and consistency
- **Enforce referential integrity at the database level.** Enable "check integrity" on every foreign key reference unless there's a specific reason not to (e.g. deliberately historical/soft references, or a reference into a view — see "References (foreign keys)" above, where `check_ref = false` is the norm since a view carries no physical FK constraint) — relying on application logic alone invites orphaned records the moment something bypasses the normal flow.
- **Don't add trace/audit columns (created/modified by + date) to a table on your own initiative.** Thinkwise ships a standard "trace fields" Thinkstore solution for exactly this purpose, and hand-modeling equivalent columns duplicates/bypasses it. Only add them if the user specifically asks for created/modified-by-and-date tracking on a table — and even then, don't just model the columns: ask the user to confirm whether they actually want manually-added columns, or would rather use the trace fields Thinkstore solution instead. Only proceed with manual columns once they've confirmed that's what they want.
- **Be deliberate about triggers.** Treat them as a last resort for enforcing data rules — look at control procedures, defaults, integrity checks, or expression columns first, since triggers hide logic outside the model and complicate generated code and debugging.

## When to use a unique index
Thinkwise's Software Factory has no separate "unique constraint" — uniqueness outside the primary key is always a **unique index** (Datamodel → Tables → Indexes → check "Unique"). **The primary key itself does not need one of these index rows at all** — it's established purely by each key column's own `primary_key` flag; verified live across a whole real model, zero rows anywhere used `indx.primary_key = true`. Don't model an `indx`/`indx_col` row just to represent a table's primary key — reserve `indx` for genuinely additional secondary/unique/non-clustered indexes.
- **Use one whenever a column (or combination) is a natural/business key that isn't the primary key** — e.g. email address, customer code, an order number within a scope — so the database rejects duplicates rather than relying on application-level checks alone.
- **Don't duplicate the primary key.** A unique index matching the PK's columns exactly is redundant, flagged by a Software Factory validation (2023.1+), and can create FK-dependency issues that make it hard to drop or regenerate.
- **Prefer a unique index over a code-level "no duplicates" check** for: consistent translated error messages from the platform (vs. raw SQL Server constraint text), and the ability to filter out NULLs so optional-but-must-be-unique-when-present columns behave correctly.
- **Regenerate and execute after adding one** — a unique index has no effect until the source is generated and executed against the database (a common cause of "it still lets me create duplicates").

## Quirks when scripting these changes through a metadata-driven modeling API

Scripting domains/tables/columns/references (and more) through a metadata/staging-style write API,
rather than the Software Factory UI directly, surfaces a set of verified-live quirks — enum keys
rejected in favor of raw numeric values, individual fields silently dropped from combined writes,
child rows with their own hidden NOT-NULL order-number fields, introspection payload size limits,
parent-resolution failures on weak entities, translatable-field writes rejected pre-commit,
unscoped queries matching the wrong model/branch, screen-type creation being UI-only, and blanket
write rejections that are actually a role/rights gap. See `references/api_write_quirks.md` for the
full list before scripting bulk changes through this kind of API.
