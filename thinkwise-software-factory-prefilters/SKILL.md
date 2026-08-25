---
name: thinkwise-software-factory-prefilters
description: Reference guide for creating and configuring prefilters and prefilter groups (tab_prefilter, tab_prefilter_grp, tab_variant_prefilter_overview) in a Thinkwise Software Factory data model — column-based vs. query-based prefilters, and prefilter groups. Use whenever an MCP connector with Software Factory access creates, inspects, or troubleshoots a prefilter or prefilter group, including before calling get_entity_definition/execute_odata_query/stage_resource against these entities.
---

# Prefilters in the Thinkwise Software Factory

A **prefilter** is a named, modeled filter condition on a table (`tab_prefilter`), optionally
bundled with sibling prefilters into a **prefilter group** (`tab_prefilter_grp`) — configured once in
the Software Factory, then offered to end users as a toggle instead of something they type into a
search box. It
renders in the Windows GUI ribbon/context menu, in the Universal UI's prefilter bar (collapsing into
an overflow menu when space runs out), and can independently scope a look-up popup narrower than the
table's own main screen.

Apply this whenever an MCP connector with Software Factory access is used to create, inspect, or
modify a prefilter or prefilter group — follow the connector's standard discovery→act flow; never
guess entity/task/property names.
`tab_prefilter`/`tab_prefilter_grp`/`tab_variant_prefilter_overview` typically live in a
`manage_datamodel`-style domain (confirmed live under that name in one connector, and identically
under a `manage_screentypes`-style domain in the same connector — check both if one comes back empty)
— try these directly first, and only escalate to `search_capabilities`/`get_available_domains` on an
`entity_set_not_found`/`domain_not_found`-style rejection rather than re-discovering a domain that
already resolved earlier this session. This skill is connector-agnostic: it names entities, fields,
and enums, not any one connector's literal tool names.

Prefilter **groups** are also an action-bar building block — how a group sits in a screen type's
action bar, and its `custom_display_type` fallback chain, isn't documented in any skill right now;
verify both live via `get_entity_definition` against `tab_prefilter_grp`/`tab_task_grp` and the
screen type's own action-bar entities rather than assuming documented behavior. This skill covers the
prefilter/group entities themselves and how to author the filter logic behind them, not the
action-bar placement.

## Plan first

Before staging the first `tab_prefilter`/`tab_prefilter_grp` row, list the proposed prefilters and
groups — name, column-based or query-based (and, for each, the column/condition/value or the actual
query fragment), which group it belongs to, match-all vs. match-any, and mandatory/exclusive — and get
the user's explicit confirmation. This is the `thinkwise_software_factory_mcp_base` "Confirm-before-
mutate" convention applied at this skill's own grain; it isn't superseded by anything below about
these entities' write mechanics. Two choices in this list are worth calling out explicitly rather than
folding silently into the plan:

- **Query-based prefilters** — the fragment runs on every grid refresh for every user who can see the
  prefilter. Walk the user through the actual SQL (or the plain-language condition it encodes) as part
  of the plan, rather than staging it and letting a wrong join or a missed edge case surface later.
- **Match-all vs. match-any** — get this wrong and the group either hides records the user expects to
  see or shows nothing at all when they combine members. It has gone wrong in a real model: see
  "Prefilter groups" below, where a live `declaration_lines.status` group uses match-all despite
  holding mutually-exclusive status values — exactly the shape the match-any guidance in this skill
  says should be match-any. Don't let "what an existing similar-looking group already uses" stand in
  for stating the intended combination semantics in the plan.

## Verified entity reference

Confirmed live against a connected Software Factory model.

**Confirmed live: both `tab_prefilter` and `tab_prefilter_grp` support the plain add/edit/commit write
flow directly** — no special creation task is needed the way some other model objects require (see the
`prog_object`-style access gaps documented in sibling skills). This is gated entirely by the
connector's own role rights, though: a write can be rejected outright even though the entity is a
plain, well-formed weak entity under `tab` and reads work fine — see the last two checklist items
below before concluding a rejection means the object isn't directly writable.

### `tab_prefilter`

| Field | Type | Notes |
|---|---|---|
| `tab_id`, `tab_prefilter_id` | key | Scoped to one table; the prefilter's own id |
| `tab_prefilter_description` | string | Translatable label |
| `prefilter_type` | enum | `query` (`0`) or `prefilter_col` (`1`) — see "Two ways to define a prefilter" below |
| `query` | string | The SQL fragment, **only populated when `prefilter_type = query`** — empty for `prefilter_col` rows (see the gap noted below) |
| `tab_prefilter_grp_id` | FK, nullable | Which group this prefilter belongs to, if any |
| `main_prefilter_state` / `detail_prefilter_state` / `look_up_prefilter_state` | enum | Independent per-context state — see "Prefilter states" below |
| `icon_id` / `icon` | — | Action-bar icon. Set `icon_id` to a suitable icon per `thinkwise_software_factory_icons` as part of creating the prefilter — it should represent the *resulting subset* (e.g. "Overdue" → clock/exclamation), never a generic funnel; don't leave it unset by default. |
| `order_no` / `abs_order_no` | int | Display order |
| `screen_area_id` | FK | Which screen area routes this prefilter's control into the layout |
| `custom_display_type` | enum | Same 9-value fallback-chain enum as `tab_prefilter_grp` and `tab_task_grp` — no skill currently documents this table's values; confirm them live via `get_entity_definition` rather than assuming a prior enum mapping still holds |
| `generated_by_control_proc_id` | FK, nullable | Set only if a control procedure owns/regenerates this row |

### `tab_prefilter_grp`

| Field | Type | Notes |
|---|---|---|
| `tab_id`, `tab_prefilter_grp_id` | key | |
| `tab_prefilter_grp_description` | string | Translatable label |
| `sub_menu` | bool | Renders the group as a dropdown instead of inline buttons. **Propose `true`/enabled as the starting point when presenting the plan for confirmation** — this is a standing user preference, not a platform default (the field itself defaults to `false` on a plain add). Apply it without asking only if the user has already stated a preference earlier in the conversation — see `thinkwise_software_factory_mcp_base`'s "Ask, don't default" convention. |
| `allow_multiple_active_prefilters` | bool | Off = at most one member active at a time (radio-style / exclusive). On = several members can be active together, combined per `filter_mode` |
| `filter_mode` | enum | `match_all` (`0`, AND) or `match_any` (`1`, OR) — only meaningful when `allow_multiple_active_prefilters = true` |
| `mand` | bool | Mandatory — the group can never end up with every member off |
| `icon_id` / `icon`, `order_no`, `custom_display_type` | — | Same shape as `tab_prefilter` — set `icon_id` to a suitable icon representing the group's shared category, per `thinkwise_software_factory_icons`. **Propose `custom_display_type` = `icon_text_text_only_icon_only_overflow` (`0`)** — "Icon + text", falling back to text-only, then icon-only, then overflow — as the starting point when presenting the plan for confirmation. Apply it without asking only if the user has already stated a preference earlier in the conversation — see `thinkwise_software_factory_mcp_base`'s "Ask, don't default" convention. Same standing default as `tab_task_grp`, see `thinkwise_software_factory_tasks`. |

### `tab_variant_prefilter_overview`

Per-`tab_variant_id` override of one prefilter's presentation and state — same
`main_prefilter_state`/`detail_prefilter_state`/`look_up_prefilter_state`/`icon_id`/`order_no`/
`screen_area_id`/`custom_display_type`/`tab_prefilter_grp_id` fields as the base `tab_prefilter`, plus
a `conditional_layout_code` field for driving the variant's presentation from a conditional-layout
expression. This is how one physical prefilter shows up differently — a different state, a different
group, even a different icon — depending on which variant's screen a user is on.

## Search, filter, prefilter, or variant?

Before modeling a prefilter, confirm it's actually the right mechanism for the need — these four
overlap in what they can technically achieve, but each fits a different user question:

| User need | Best starting mechanism |
|---|---|
| "I know part of the name/code/reference" | Search (`col.visible_for_search`, see `thinkwise_datamodeling_guidelines`) |
| "Status is Open and due date is before Friday" | Filter (`col.visible_for_filter`, same skill) |
| "My open work," used every day | Prefilter |
| "Planning view for dispatchers with different columns and sort" | Variant (`thinkwise_software_factory_variants`) |
| "One selected lookup value" | Column-header/quick filter |
| "Several complex, reusable conditions" | Prefilter, possibly user-defined |
| "Different authorization or business semantics" | Role plus variant/subject design — not just a filter |

Reach for a prefilter specifically for **named, recurring subsets** users recognize by label — *My
work*, *Overdue*, *Unprocessed*, *Errors*, *Active* — not a bar holding one toggle per possible status
value. Prefer a few high-value shortcuts and let the ordinary filter popup handle uncommon
combinations; see `thinkwise_datamodeling_guidelines`'s `references/subject_presentation_design.md`
for the fuller sort/search/filter design reasoning this table summarizes.

## Two ways to define a prefilter

Set on the prefilter's Form tab (`prefilter_type`):

**Prefilter columns** (`prefilter_col`) — pick a **Column**, a **Filter condition**, and a **Filter
value**; no SQL. Add more than one column and they combine with AND inside that one prefilter. The
filter value, when the column is typed to a domain, is the domain element's raw **database value** —
never its **ID/name** (e.g. `approved`) and never its **translated caption** (e.g. "Approved"); see
`thinkwise_datamodeling_guidelines`'s "Domain elements" section for the value/ID/translation distinction.
This reuses the same condition
vocabulary as the column's own filter/search settings (`col.filter_condition` enum, confirmed live):

| Value | Condition | | Value | Condition |
|---|---|---|---|---|
| 0 | Equal to | | 9 | Does not contain |
| 1 | Not equal to | | 10 | Is empty |
| 2 | Greater than | | 11 | Is not empty |
| 3 | Smaller than | | 12 | Does not start with |
| 4 | Greater than or equal to | | 13 | Not between |
| 5 | Smaller than or equal to | | 14 | Ends with |
| 6 | Between | | 15 | Does not end with |
| 7 | Starts with | | 16 | In |
| 8 | Contains | | 17 | Not in |

**Query** (`query`) — a hand-written SQL fragment, inserted straight into the generated `where`
clause, aliased `t1` for the table's own row (see "Query structure" below). Use it once the condition
needs more than "this column compares to that value": a join, an `exists` subquery, a function call,
or a value that isn't static (the current user, `getdate()`, and so on).

**Prefer a prefilter column whenever the shape allows it.** It needs no SQL to write or review, is
validated by the Software Factory itself (a renamed or retyped column is caught immediately, unlike a
query fragment which silently keeps referencing the old name), and stays consistent with the table's
own filter/search conditions. Reach for a query only when the condition genuinely can't be expressed
as a column comparison to a fixed value.

**Gap, verified live**: the Prefilter columns grid (Column / Filter condition / Filter value rows
behind a `prefilter_col`-typed prefilter) is **not exposed as a plain entity set** in either domain
checked — `tab_prefilter.query` comes back empty for every `prefilter_col` row, and no
`tab_prefilter_col`-style child entity set exists alongside it. This is the same access-gap pattern as
`screen_component` in the screentypes skill: the data exists in the model (the Software Factory's own
UI reads and writes it) but isn't surfaced as ordinary CRUD rows through the standard domain metadata
inspected here.

- **Check the connector's own domain metadata first** — `search_domain_capabilities` with keywords
  like `"prefilter column"`/`"filter condition"`/`"filter value"` — in case a different connector or a
  newer domain does expose it as a first-class entity set.
- **If it doesn't**: extend the connector's domain metadata to reach the underlying (likely
  modeler-specific) rows, the same first-choice path the screentypes skill recommends for
  `screen_component`; or, more simply, **model the equivalent condition as `prefilter_type = query`**
  instead — a Column/Condition/Value triple like `status` / Equal to / `2` becomes the query fragment
  `t1.status = 2`. This is functionally equivalent for the end user (same result set, same states,
  same grouping), the only loss is the Software Factory's own live validation of the column reference
  and the column-condition UI in its Prefilter columns grid.

## Query structure

A query-based prefilter is a boolean SQL expression dropped into the table's `where` clause. The
current row is always aliased `t1`:

```sql
t1.order_date < getdate()
```

**Verified: `tab_prefilter` has no `rdbms_type` dimension** — unlike `control_proc_template`, which is
keyed per dialect (see `thinkwise_software_factory_create_control_procedures`), there is exactly one
`query` string per prefilter, applied against every enabled RDBMS. If `branch_rdbms_type` for the
model has more than one row, write portable SQL (avoid dialect-specific functions —
`getdate()`/`now()`/`sysdate`, `top n`/`limit`, string-concat operators) or restructure the condition
so it doesn't need one — there is no per-dialect override slot the way a template-backed view or
control procedure has.

Four real examples, confirmed against a live reference model:

**Relative date, via a calendar table** (`hour_analysis.current_week`):
```sql
exists (
    select 1
      from date_helper dh
     where dh.iso_date = cast(getdate() as date)
       and dh.iso_week = t1.week
       and year(iso_date) = t1.year
   )
```
Comparing to `getdate()` rules out a plain column condition — the value isn't static.

**"My records", via the current user** (`email.my_emails_employee`):
```sql
exists (
    select 1
      from application_settings a
     where a.standard_employee_id = t1.employee_id
   )
```
A per-user settings table keyed off a `dbo.tsf_user()`-style function is the standard way to reach
"the logged-in user" from SQL. The same shape repeats for `my_customers`, `my_declaration`,
`my_emails_contact` — always an `exists` join through the same per-user settings table.

**Data-quality check, via a correlated count** (`activity.untranslated`):
```sql
8 <> (select count(1) from activity_translated t where t1.activity_id = t.activity_id)
```
`8` is the number of active application languages; the prefilter flags any row missing a translation
row for one of them. Generalizes to any "should have exactly N related rows" completeness check.

## Prefilter states

Every prefilter carries three independent states — `main_prefilter_state`, `detail_prefilter_state`,
`look_up_prefilter_state` — so the same prefilter can behave differently on the main grid, a detail
grid, and a reference field's popup. Confirmed enum, all three fields:

| State | Value | Visible | Active | User can change it |
|---|---|---|---|---|
| Off hidden | 0 | No | No | No |
| Off | 1 | Yes | No | Yes — can turn on |
| On | 2 | Yes | Yes | Yes — can turn off |
| On locked | 3 | Yes | Yes | No |
| On hidden | 4 | No | Yes | No |

`On hidden` is how a prefilter becomes a silent data-authorization filter — the condition always
applies and the user never sees the control. Use `look_up_prefilter_state` on its own (leaving
Main/Detail at Off) to narrow a reference field's popup — e.g. a project picker offering only *active*
projects — without touching what the table shows on its own screen.

**A table variant may only tighten a prefilter's state relative to the base table** (On → On hidden is
fine; On hidden → Off is rejected) — documented platform behavior, not independently re-verified
against `tab_variant_prefilter_overview` validation in this pass. If a genuinely looser state is
needed somewhere, model a second variant rather than fighting the check.

**Role-based "Always on" / data-authorization** can force a prefilter to `On hidden` for a specific
role regardless of what the active variant says (Access control → Model rights → Tables → Prefilters →
Roles, per platform docs). The role/rights entity backing this was **not found** among the
`manage_datamodel`/`manage_screentypes`-style domains checked here — none of the domains a connector
exposed included a `role_tab_prefilter_overview`-shaped entity. Check the connector's own domain list
for a security/IAM/rights-style domain before assuming this is unreachable through it.

## Prefilter groups

`allow_multiple_active_prefilters = false` makes a group behave like a radio button: zero or one
member active at a time. Setting it `true` allows several members active together, combined per
`filter_mode`:

- **Match all (AND)** — a row must satisfy every currently-active prefilter in the group.
- **Match any (OR)** — a row must satisfy at least one active prefilter in the group. **Universal UI
  only** — the Windows GUI keeps at most one active prefilter per group no matter this setting.

Thinkwise's own illustrative case for match-any: a status column with Draft / Active / Paid / Payment
rejected prefilters in one group. Turn on Draft and Active together and you see records matching
*either* — AND would return nothing, since a row can't hold two status values at once. This is the
general rule for any group of mutually-exclusive category values: **match any**, not match all.

A **mandatory** match-any group is also the standard way to build category-level authorization: put
Red/Green/Blue in a mandatory OR group, then set Red to `Off hidden` for roles that shouldn't see red
records. Turning a member on in an OR group *widens* the result set rather than narrowing it — so
`Off hidden` there means "this role's Red toggle doesn't exist," not "Red data is blocked" — a common
point of confusion when authorization is layered onto an OR group.

Two groups observed in a live model, for contrast — the first is the same `declaration_lines.status`
match-all anomaly called out in "Plan first" above (see there for the full explanation):

| Table | Group | Multiple active | Mode | Members |
|---|---|---|---|---|
| `declaration_lines` | `status` | Yes | Match all | `approved`, `cancelled`, `paid`, `to_be_approved` |
| `email` | `read_or_unread` | No | — | `read`, `unread` |

## Common patterns

Six worked examples (status checklist, exclusive two-state toggle, "my records", relative date
window, look-up-only scoping, data-quality flag) — see `references/prefilter_patterns.md` for the
full write-up of each.

## Translation

`tab_prefilter_description` and `tab_prefilter_grp_description` are translatable labels like any
other model text — see `thinkwise_software_factory_translation_objects` for the full mechanics
(`transl_object`/`transl_object_transl`, the bracket-placeholder default, `branch_appl_lang`). Setting
the `_description` field when creating a prefilter or group only fills the base/default language;
that's the same one-language-at-a-time trap the translation skill describes for every other object.
Before considering a prefilter or group done, check `transl_object_transl` across every language the
branch actually supports (`branch_appl_lang`) for lingering `[bracket]`-placeholder text on that
object, rather than assuming one write covered every configured language. Confirm the live
`type_of_object` value for `tab_prefilter`/`tab_prefilter_grp` rather than guessing it — the
translation skill deliberately doesn't hardcode this enum, since it spans roughly 150 concepts and
grows across platform versions. Confirmed live in one model: `prefilter` = `9`, `tab_prefilter_grp`
= `15` — re-verify rather than trusting these across connectors/versions.

## Pre-flight checklist

- **Propose `sub_menu` (dropdown vs. inline buttons) = `true`/enabled, and `custom_display_type` =
  `icon_text_text_only_icon_only_overflow` (`0`), as the starting point for a new prefilter group when
  presenting the plan for confirmation.** Both are standing user preferences, not the field's own
  platform default — apply either without asking only if the user has already stated a preference
  earlier in the conversation (`thinkwise_software_factory_mcp_base`'s "Ask, don't default"
  convention).
- **Default to a prefilter column; write a query only when the shape demands it** — a join, a
  subquery, a function call, or a dynamic value (current user, current date) is the signal a query is
  actually needed. A plain "column equals value" never is.
- **Set `icon_id` on every new prefilter/group to a suitable icon** representing the resulting subset,
  per `thinkwise_software_factory_icons` — don't leave it unset by default.
- **Set a tooltip on every new prefilter (and prefilter group) as part of finishing it** — a Software
  Factory validation flags a task, report, or prefilter left with no tooltip text at all.
- **Filter value is the domain element's database value** — never its ID/name (e.g. `approved`) and
  never its translated caption (e.g. "Approved") — same rule as any other filter/search condition on
  that column.
- **Check for a `tab_prefilter_col`-style entity on the connector before assuming column-based
  prefilters can't be authored through the API.** It wasn't found in the domains checked here; if it's
  also absent on the connector in use, fall back to `prefilter_type = query` and write the equivalent
  `where`-clause fragment directly.
- **`tab_prefilter.query` has no `rdbms_type` dimension.** Check `branch_rdbms_type` before writing a
  query-based prefilter — more than one row means the SQL must be portable across every enabled
  dialect, since there's no per-dialect override slot here.
- **A variant can only tighten a prefilter's state, never loosen it.** Model a second variant if a
  genuinely looser state is needed elsewhere.
- **Match any (OR) is Universal UI only** — don't rely on it for a Windows GUI client; at most one
  prefilter in a group is ever active there.
- **Mutually-exclusive category values usually want match-any, not match-all** — verify the intended
  combination semantics explicitly rather than copying whatever an existing similar-looking group in
  the model already uses.
- **`On hidden` + role rights is the authorization pattern** — a hidden prefilter the user can't see or
  clear, forced per role, is how prefilters double as row-level security. If the connector doesn't
  expose a role/rights domain, say so rather than assuming the capability doesn't exist at all.
- **Keep query-based prefilters lean** — they run on every grid refresh; analyze anything joined into
  one for performance and keep heavy views or high-volume tables out of it.
- Confirm the connector's actual domain key for prefilters (`manage_datamodel`-style, sometimes
  mirrored under a `manage_screentypes`-style domain too) before assuming one fixed name holds
  everywhere.
- **Patch one field at a time when setting up a new prefilter or group, and verify each response
  before moving on** — another instance of the general "a multi-field write can silently drop one
  field" behavior in `thinkwise_datamodeling_guidelines`'s quirks section. Passing several properties
  together in one ordered batch has intermittently
  reset one of them back to its prior value in the response — not consistently the same field, not
  consistently the first or last in the list — even though nothing in the request indicated a
  rejection. Batching is fine to attempt, but read back the field you actually care about (especially
  `tab_prefilter_grp_id` and `query`) before committing, and re-patch it alone if it didn't stick.
- **A write rejection on one entity set doesn't mean the branch or session is blocked wholesale.**
  Software Factory can grant write rights per entity set independently of branch state — a rejected
  add/edit on `tab_prefilter`/`tab_prefilter_grp` can coexist with working writes to an unrelated
  entity under the same parent (a plain column, for instance). Test that unrelated write first to
  isolate an entity-specific rights gap before concluding the branch or connector needs
  reconfiguring. Note also that a branch's own `protected` flag is not itself editable through this
  write flow (an edit attempt on the `branch` record is rejected the same way) — toggling it requires
  the Software Factory's own UI directly.
- **Prefilters and prefilter groups are translatable objects** — a `_description` write only fills the
  base/default language; check every other configured branch language for lingering
  `[bracket]`-placeholder text before considering a prefilter or group done (see
  `thinkwise_software_factory_translation_objects`).
