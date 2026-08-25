---
name: thinkwise-software-factory-tasks
description: Reference guide for creating and configuring tasks in a Thinkwise Software Factory model — task types, parameters, look-ups, form setup, variants, and table task assignment. Use whenever an MCP connector with Software Factory access creates, inspects, or troubleshoots a task, before working with the task, task_parmtr, task_ref, task_variant, or tab_task family of entities.
---

# Creating Tasks in the Thinkwise Software Factory

A **task** is the Software Factory's unit of "do something" — anything from a single UPDATE
statement to a call out to an external program. Every task is its own top-level object (`task`,
keyed only by `task_id`), independent of any table. It only becomes a **table task** once it's
bound to one via `tab_task`; until then it's an **unbound** task, reachable only from a menu item,
a process flow, or the API.

Apply this whenever an MCP connector with Software Factory access is used to create, assign, or
inspect a task — follow the connector's standard discovery→act flow; never guess entity/task/property
names. Everything below was confirmed live against a connected model via `sf_mcp` (domain key
`sf/manage_tasks`) — try that domain directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session. This skill
is connector-agnostic: it names entities, tasks, and fields, not any one connector's literal tool
names.

**Companion skills, not duplicated here**: the actual control-procedure/template mechanics behind
a task's generated code (code groups, business-logic variables, static vs. SQL assignment, the
two-step "generate code group then generate object code" sequence, multi-RDBMS dialects) live in
`thinkwise_software_factory_create_control_procedures` — read it before writing a task's template.
Full translation mechanics (`transl_object`/`transl_object_transl`, `type_of_object`, approval
workflow) live in `thinkwise_software_factory_translation_objects`. This skill covers what's
specific to tasks; both companions cover the shared machinery in depth.

**Gotcha, confirmed live**: `tab_task`, `task_parmtr`, `tab_task_grp`, and `tab_task_parmtr` are also
reachable as entity sets under a `manage_datamodel`-style domain — but the master `task`,
`task_variant`, and `task_ref`/`task_ref_col` entities were **not** found there
(`entity_set_not_found`) and only resolved under `sf/manage_tasks`. Don't assume every task-adjacent
entity lives in the same domain as the rest of the data model — confirm each one via
`search_domain_capabilities`/`get_available_domains` rather than guessing from where a sibling entity
happened to resolve.

## Plan first

This is the task-grain application of `thinkwise_software_factory_mcp_base`'s "Confirm-before-mutate"
convention (see its Shared conventions section) — don't restate that rule, apply it. Before the first
`stage_resource`/`stage_task` call for a new or changed task, state the plan and get it confirmed. For
a task, "the plan" means naming:

- The **task type** (`task_type_id`) and whether it's **bound or unbound** — and if bound, to which
  table.
- The full **parameter list** — each parameter's direction (input/output) and purpose.
- Any **look-ups** (`task_ref`) the parameters need, and what they point at.
- Which of the **four form mechanisms** — Groups / Conditional layout / Layout / Defaults (see "Form
  setup" below) — the form will use, and why.

This generalizes the `task_conditional_layout`-specific confirmation rule further down (never add one
without checking first) to the whole task, not just its conditional layout. If the task is part of a
larger build (e.g. from `thinkwise_software_factory_build_planner`), fold this into that plan instead
of confirming it separately once the task is already being staged.

## Golden rule — creation order is enforced, not just conventional

A brand-new task's pieces have to be created in this order; the API rejects several of the
shortcuts:

1. **The `task` row first** — a plain `stage_resource`(add)/`patch_resource`/`commit_resource`
   against `task`, keyed only by `task_id`. There is no dedicated "create task" bound task; it's an
   ordinary entity add, the same as creating a `tab` or `col`. Nothing else below can exist before
   this row does.
2. **Bind it to a table via `tab_task`, if it's a table task** — keyed by `(tab_id, task_id)`.
   Setting `tab_task.task_id` to a `task_id` that doesn't exist yet is rejected outright, even
   though the field presents as an ordinary editable string — it's enforcing that the `task` row
   exists first.
3. **Add its parameters via `task_parmtr`** — keyed by `(task_id, task_parmtr_id)`, a child of
   `task`, **not** of `tab_task`. If a parent-based add doesn't resolve a navigation to `task`,
   fall back to a plain add with the full key supplied as explicit fields.
4. Only once all three exist does the control-procedure flow apply: **Generate code group** to
   materialize the `task_<task_id>` placeholder, then write/assign a template, then generate the
   actual code. See `thinkwise_software_factory_create_control_procedures` for that sequence in
   full — it applies to a task's own logic exactly as it does to a table's.

**A table task does not receive that table's primary key for free.** Unlike a Handler, which gets
the row's key automatically, a task bound via `tab_task` has no built-in way to know which row it's
acting on. If the task's logic needs the row's identity, add an explicit `task_parmtr` matching the
table's PK column id (e.g. `lead_id` for a task on `lead`) — otherwise this doesn't surface as an
error until the logic is written and something is silently missing.

## The object graph

| Entity | Key | Role |
|---|---|---|
| `task` | `task_id` | The master object: logic type, confirmation, badge, communication mode. |
| `task_parmtr` | `task_id, task_parmtr_id` | One row per input/output parameter, plus its default form placement. |
| `task_ref` / `task_ref_col` | `task_ref_id` / `+tab_id, col_id` | A custom look-up config for one or more parameters. |
| `task_variant` / `task_variant_parmtr` | `+task_variant_id` | An alternate presentation of the same task and parameter set. |
| `task_conditional_layout` | `task_id, conditional_layout_id` | Static, condition-driven font/colour styling on a parameter. |
| `tab_task` / `tab_task_parmtr` | `tab_id, task_id` / `+col_id, task_parmtr_id` | Binds the task to a table; binds a parameter to a column. |
| `tab_task_grp` | `tab_id, tab_task_grp_id` | Groups related table tasks under one action-bar button. |
| `tab_variant_task_overview` | `tab_id, tab_variant_id, task_id` | Per table-variant override — which task variant shows, its ordering/visibility. |
| `task_variant_look_up_overview` | `task_id, task_variant_id, task_ref_id` | Per-variant override of a look-up. |

## Task logic types (`task.task_type_id`)

This decides what kind of program object the task actually generates:

| task_type_id | Shown as | When to use it |
|---|---|---|
| `STORED_PROCEDURE` | Template | The default for real server-side logic. Generates an actual stored procedure from a control-procedure Template — everything in `thinkwise_software_factory_create_control_procedures` applies to this type. **Only this type has a template to assign.** |
| `FUNCTION` | GUI code | Client-side logic that never touches the database — copy-to-clipboard, open a URL, trigger a download. No control-procedure template; the logic lives in GUI code instead. |
| `EXTERNAL_PROCEDURE` | External stored procedure | Calls a stored procedure that already exists in the database, outside anything the Software Factory generates. |
| `EXTERNAL_FUNCTION` | External function | Same idea, for a database function. |
| `EXTERNAL_PROGRAM` | Windows command | Launches an OS-level command/executable from the server. |
| `ITP` | ITP interface | Legacy ITP integration task type. |
| `DUMMY` | None | No generated program object at all — a pure carrier (menu entry point, or something that only passes parameters into a process flow) with no server logic of its own. |

Before hunting for a missing Assigning-screen entry, check `task_type_id`: only `STORED_PROCEDURE`
tasks (and their Default/Layout/Badge sub-objects, when enabled) go through the control-procedure
assignment flow.

**Confirmed live**: setting `task_type_id=DUMMY` makes the task's own `object_name` field flip to
hidden/non-editable (there's no generated object to name) — don't try to patch it for a `DUMMY` task;
only types that actually generate something (`STORED_PROCEDURE`, and presumably the other
non-`DUMMY`/non-`ITP` types) need/accept `object_name`.

`task.type_of_communication` ("Await result") is the separate, second setting controlling how the
GUI waits for the task: `synchronous` (0), `async` (1), `synchronous_with_progress` (2, the default
for Template tasks), `synchronous_with_progress_possible_async` (3).

## Task-level row &amp; behavior settings (`task` entity, verified live)

Beyond `task_type_id` and `type_of_communication` above, these `task` fields govern how the task
behaves once triggered — confirmed live against `sf/manage_tasks`:

| Field | Shown as | When to set it |
|---|---|---|
| `single_transaction` | Atomic transaction | On for a Template task whose related data changes must succeed or roll back together. |
| `ask_confirmation` / `confirmation_msg_id` | Ask confirmation | On for destructive/irreversible/expensive/high-impact actions, with a specific message — see `thinkwise_software_factory_messages`'s "Task confirmation messages". |
| `popup_for_each_row` | Popup for each row | Off (the default) to show the task's form/confirmation once for the whole selection; on only when each selected row genuinely needs its own input or its own confirmation — flipping it on for an ordinary bulk action is the message-storm anti-pattern (`thinkwise_software_factory_messages`). |
| `shift_code` + `ascii_code` | Shortcut | Together form the keyboard shortcut (a modifier plus a key code) — set both or neither; avoid GUI/reserved or duplicate combinations. |
| `show_badge` + `badge_interval` | Show badge / Badge interval (seconds) | Only when a meaningful badge query will actually be implemented and refreshed at a sensible interval — see the Badge concept in `thinkwise_software_factory_create_control_procedures`. |
| `repeat_after_execute` | Repeat after execute | On for repetitive capture workflows (e.g. scanning) where the form should reopen immediately after each execution, not for ordinary one-shot actions. |
| `generation_order_no` | Generation order | Change only when one generated task genuinely depends on another having generated first. |
| `offline_executable` | Offline executable | On only after checking platform constraints — an offline task cannot use look-ups, and any parameter meant to be filled by the user (rather than an offline-safe default) must be hidden. |
| `icon_id` | Icon | Set to a suitable icon from the repository as part of creating the task — the task's own icon wherever it appears (action bars, menus). Follow `thinkwise_software_factory_icons`; a task icon should show the outcome/verb, not a generic gear. |

## Bound, unbound &amp; table tasks — and the security implication

An unbound task (no `tab_task` row) can still be a menu item, a process-flow step, or an IAM
start object — and per platform-team guidance, **it remains reachable through the Indicium API even
with none of those wired up.** Not showing a task anywhere in the GUI is not the same as securing
it; role/rights configuration is what actually restricts who can call it. If logic should never be
independently callable at all, prefer a genuine subroutine/function control procedure over an
unassigned task.

A **table task** is the same `task` row plus a `tab_task` binding — that's what gives it row
context and lets `tab_task_parmtr` auto-fill a parameter from the current row's column (see
Parameters below).

## Parameters (`task_parmtr`)

Each row is one input or output on the task's form, independent of any table:

| Field | Purpose |
|---|---|
| `task_input` / `task_output` | Direction: typed in by the caller, or returned for the caller to use afterward. |
| `type_of_col` | `editable` (0) / `read_only` (1) / `hidden` (3) — static, independent of the dynamic Layout mechanism below. |
| `mand` | Static mandatory-ness, before any dynamic Layout logic can loosen or tighten it. |
| `dom_id` | Domain backing the parameter's type/control — set explicitly, don't trust a default. |
| `type_of_default_value` / `default_value` / `default_value_query` | A literal default, or a query evaluated when the form opens — the lightweight alternative to a full Default control procedure. |
| `case_type` | Upper / lower / initial caps / proper case, enforced on input. |
| `alias_task_parmtr_id` | Points this parameter's identity at another parameter's — the same borrowing pattern `col.alt_transl_col_id` uses for columns, for deliberately sharing presentation/translation rather than duplicating it. |
| `label_width` / `field_width` / `field_height_in_positions` / `field_no_of_positions_further` / `field_in_next_col` | Static form placement/sizing — pure layout, no code. |
| `form_field_in_next_grp` + `form_next_grp_label` | Starts a new labelled group on the form. |
| `field_on_next_tab` + `next_tab_label` | Starts a new section/tab on the form. |
| `layout_input` / `layout_type_output` / `layout_mand_output` | Column-level gate for the Layout concept (see Form setup). |
| `default_input` / `default_output` | Column-level gate for the Defaults concept (see Form setup). |

**Table task parameter binding**: `tab_task_parmtr` (keyed by `tab_id, task_id, col_id,
task_parmtr_id`) binds a parameter to a specific column of the bound table so it auto-fills from
the selected row instead of asking the user to type it. This is the explicit step needed to give a
table task the row's PK or any other row-derived value — nothing does this automatically.

## Task look-ups (`task_ref` / `task_ref_col`)

Reach for a custom look-up whenever a parameter's own domain doesn't already point at the right
table, or the default look-up behaviour isn't what's wanted:

- A different table, or a specific **table variant** (`look_up_tab_variant_id`) rather than the
  default one.
- A different **display column** (`look_up_display_col_id`) than the table's own default.
- A different **look-up control**: `auto_complete` (0), `combo_alphabetical` (1),
  `combo_sorted` (4), `suggestion_contains` (2), `suggestion_starts_with` (3).
- A **popup picker** (`look_up_has_popup`) instead of an inline combo/suggestion field.
- A **composite** look-up spanning multiple columns — `task_ref_col` maps each contributing
  `(tab_id, col_id)` to the parameter(s) it feeds, in order (up to nine columns via the
  `task_create_task_ref` bound task's `col_id_1..9`/`task_parmtr_id_1..9` parameters).

Create one either standalone against `task_ref` (+ a `task_ref_col` child row per contributing
column), or directly from the owning task via `task_create_task_ref` (bound to `task`) — in principle
both take the same shape. **`task_create_task_ref` proved unreliable in practice, verified live**: staging
it succeeded, but the very next call against the returned staged resource failed with a
"type not found"-style rejection, even though nothing about the request looked malformed. The
standalone `task_ref`/`task_ref_col` add flow for the identical look-up succeeded without issue.
Default to the standalone flow rather than the bound task, and treat `task_create_task_ref` as
suspect if it's ever reached for again. `task_modify_task_ref` edits a look-up afterward. Per-task-variant
overrides live in `task_variant_look_up_overview`, with its own reset-to-base action
(`task_reset_task_variant_look_up_overview`).

## Form setup: four mechanisms, easy to conflate

They differ on two axes — **static vs. dynamic**, and **cosmetic vs. behavioural**:

| Mechanism | Static/dynamic | What it does | Reach for it when… |
|---|---|---|---|
| **Groups** (`form_next_grp_label`/`next_tab_label` on `task_parmtr`) | Static | Pure visual grouping/sectioning. No logic. | The form just needs organizing into labelled sections — always the first tool. |
| **Conditional layout** (`task_conditional_layout`) | Static, condition-gated | No-code font/colour styling triggered when a stated condition on a parameter is true. | The need is purely cosmetic emphasis (e.g. highlight a value past a threshold), not a real show/hide/mandatory change. |
| **Layout concept** (`task.use_layouts` + a Layout control procedure) | Dynamic (runtime SQL) | Real-time control of a parameter's visibility (`@[parmtr]_type`: normal/read-only/hidden-in-form/hidden-outside-form) and mandatory-ness, plus Confirm/Cancel button types — reacts to `@layout_mode`/`@cursor_from_col_id` and other parameters' current values. | Fields must actually appear, disappear, or become mandatory based on other input on the same form. |
| **Defaults concept** (`task.use_defaults` + a Default control procedure) | Dynamic (runtime SQL) | Computes a parameter's value — once on open, or reactively per edit — via `@default_mode`/`@cursor_from_col_id`. | A parameter's value should be derived, not typed (a constant or `default_value_query` on the parameter itself is the lighter option for trivial logic). |

**Don't pick unilaterally when it's a close call.** This table is for the clear-cut cases. If it's not
obvious from the request which mechanism(s) actually apply — e.g. it could plausibly be cosmetic
Conditional layout or a real behavioural Layout change — ask the user rather than choosing on their
behalf; this is exactly the kind of choice "Plan first" above should surface before any staging call.

The Task code type reuses the **same** Default/Layout business-logic variables tables/rows do, just
scoped to a task parameter instead of a column (`@[task_parmtr_id]` in place of `@[col_id]`) — see
`thinkwise_software_factory_create_control_procedures`'s `references/code_type_variables.md` for
the exact per-direction variable names before writing the template body.

### Consider `task_conditional_layout` for new parameters — but only where it's warranted

After adding a task's parameters, take one pass asking whether **conditional layout** (cosmetic
font/colour styling on a parameter, triggered by a condition on it or another parameter — see the table
above) would genuinely help this specific form: a parameter whose value can exceed a threshold, a
destructive option that's been selected, a required field still empty, a date outside a sensible
window. **Only add one where there's a real candidate — don't add one just because a task has
parameters.** Plenty of tasks (a plain confirm/cancel action, a simple lookup-and-submit form) have
nothing worth highlighting; say so and add nothing rather than inventing a marginal one.

**Never add a `task_conditional_layout` without checking with the user first** — present the candidate
parameter(s), the condition, and what it would communicate, and get explicit confirmation before
creating anything. If this task is part of a larger plan (e.g. from `thinkwise_software_factory_build_planner`),
fold the candidate into that plan and get the **plan** confirmed before finalizing it, not as a silent
addendum once the task is being built.

For the actual mechanics — field reference, the condition enum, `task_variant_task_conditional_layout`
per-variant overrides, and known gaps — see `thinkwise_software_factory_conditional_layouts`. This
section only decides *whether* one is warranted; that skill covers *how* to build it.

**Both gates, every time**: the task-level `use_layouts`/`use_defaults` flag on `task` *and* the
parameter-level `layout_input`/`layout_type_output`/`layout_mand_output` or
`default_input`/`default_output` flags on `task_parmtr` must be on. An assigned template with
either gate off generates without error and simply never fires — the exact same trap the
control-procedures skill documents for table columns applies here.

## Task variants (`task_variant`)

An alternate **presentation** of the same task — same `task_id`, same parameter set, same
generated logic — with its own icon, badge, confirmation message, display parameter, and
Confirm/Cancel button labels/translations. `task_variant_parmtr` lets a variant override a
parameter's mandatory-ness or default value; `task_variant_look_up_overview` lets it override a
parameter's look-up. Both support resetting back to the base task's configuration.

A variant **cannot** change the parameter list — that needs a genuinely different `task_id`.
Variants are for "same operation, different face" — e.g. one boolean-flip task exposed as an
"Activate" variant (default value `1`) and a "Deactivate" variant (default value `0`).

Create one from the owning task with `task_create_task_variant` (parameters: `task_id`,
`task_variant_id`, `task_variant_description`, `generate_transl_object`) — leave
`generate_transl_object` on unless the variant is deliberately meant to share the parent task's
translation.

**One task, one slot per table task list.** `tab_task` is keyed by `(tab_id, task_id)` — not by
variant. A table can only carry one `tab_task` row per task, so two variants of the same task
cannot both be added to the same table's plain task list at once. The per-variant choice of *which*
`task_variant_id` shows lives one level down, on `tab_variant_task_overview` — showing two variants
side by side on the same table means giving them separate **table variants**, each independently
selecting its own task variant via that overview entity, not two `tab_task` rows.

## Assigning table tasks

### By hand

Add a `tab_task` row picking the `task_id`, then configure its table-specific presentation:
`show_tab_task`, `icon`, `order_no`, `tab_task_grp_id` (to fold it into a task group on the action
bar), `screen_area_id`, `custom_display_type` (icon/text fallback chain — see the enum on
`tab_task`/`tab_task_grp`), `primary_action`, `refresh_after_execute`
(`none`/`row`/`subject`/`document`), grid double-click behaviour
(`grid_double_click`/`grid_double_click_col_id`), and `enable_tab_task_when_empty`. Then add
`tab_task_parmtr` rows for any parameter that should auto-fill from a column of the current row.

**`tab_task.icon` is a table-specific *override*, not the task's primary icon** — leave it unset to
inherit `task.icon_id`, and only set it when this table's context genuinely warrants a different icon
than the task shows everywhere else. **It's also verified upload-only**: unlike `task.icon_id`,
`tab_task` has no `icon_id` field at all — it's a plain file upload straight onto the row, not a pick
from the shared repository, so it can't be reused or consolidated via `task_update_icon_usage`. See
`thinkwise_software_factory_icons`.

**Set `enable_tab_task_when_empty=false`** for a task whose entire purpose is acting on one specific
selected row — e.g. a task bound only to launch a process flow via `grid_double_click`. It defaults to
`true`, but a task that just shows/filters detail for "the selected row" is meaningless with no row
selected, so leaving the default on makes it reachable in a state that can't do anything useful.

Use this when the assignment is genuinely one-off, or the target table only needs a subset of an
existing table's task list.

**When a new `tab_task_grp` (task group) is called for, propose these as its defaults while
confirming the plan** — per `thinkwise_software_factory_mcp_base`'s "Ask, don't default" convention,
this is what to put forward for sign-off, not something to apply silently unless the user corrects it:
- **`sub_menu = true`** — render the group as a dropdown rather than inline buttons.
- **`custom_display_type = icon_text_text_only_icon_only_overflow`** (value `0`) — "Icon + text",
  falling back to text-only, then icon-only, then overflow, as space runs out. Both fields default to
  `false`/unset on a plain add, so set them explicitly once confirmed rather than leaving the field
  default in place. The same `custom_display_type` default applies to a `tab_prefilter_grp` — see
  `thinkwise_software_factory_prefilters`.

### Bulk-copying an existing assignment

| Task | Bound to | Copies | Use when |
|---|---|---|---|
| `task_copy_task` | `task` | The whole task; optionally its table-task assignments (`copy_object_task`) and its functionality/template assignments (`copy_object_assignment`) | Duplicating a task already wired to several tables, wanting the clone wired the same way. |
| `task_copy_tab` | `tab` | A whole table's setup onto another table — refs, look-ups, GUI, reports, and (via `copy_object_task`) its full table-task list | A new table should start with the same task/report/reference lineup as an existing, structurally similar table. |

Bulk-copy then prune is usually faster than hand-assembling a large task list — but it copies
things you may not want, so treat it as a starting point on a structurally similar table, not a
shortcut on a dissimilar one. For a small, deliberate assignment, doing it by hand is just as fast
and leaves nothing to clean up.

## Renaming, copying, deleting

Go through the dedicated bound tasks rather than editing `task_id`/`task_variant_id` in place —
keys aren't renamable directly, the same immutable-key pattern as tables/columns elsewhere in the
model:

- `task_rename_task` (`from_task_id`, `to_task_id`) / `task_delete_task`
- `task_rename_task_variant` (`task_id`, `from_task_variant_id`, `to_task_variant_id`) /
  `task_delete_task_variant`
- `task_copy_task_variant` (`from_task_id`, `from_task_variant_id`, `to_task_variant_id`)
- `task_delete_task_ref` / `task_delete_task_conditional_layout`

## Control procedures for a task's own logic

For a `STORED_PROCEDURE`-typed task, the real logic is a control procedure exactly like any other
generated object — the **TASKS** code group, plus **DEFAULTS**/**LAYOUTS**/**BADGES** scoped to the
task wherever those concepts are enabled. Object naming carries straight over: `task_<task_id>` is
the task's own executable object; `default_<task_id>`, `layout_<task_id>`, `badge_<task_id>` are
its Default/Layout/Badge sub-objects. Follow
`thinkwise_software_factory_create_control_procedures` for the full sequence — in short, once the
`task`/`tab_task`/`task_parmtr` rows exist (Golden rule above):

1. Query `branch_rdbms_type` before writing a line of SQL.
2. Run **Generate code group** (`task_generate_code_grp`, bound to any `control_proc` in the TASKS
   group) to materialize the `task_<task_id>` placeholder — nothing to assign a template to exists
   before this.
3. Write the template and assign it (Static via the Assigning screen is the natural default for one
   task's own logic; SQL/dynamic only if the same template genuinely fans out across many tasks).
4. Queue actual generation with `task_add_job_to_generate_object_code` — the code-group step above
   only creates the placeholder, this is what actually produces `prog_object_generated_code`.
5. Confirm by re-reading the object (`generated_code_stale = false`) and reading the generated text
   itself to confirm the assigned template's own logic is really in it.

`FUNCTION`, `EXTERNAL_*`, and `ITP` tasks have no template to assign — their logic lives outside
the Software Factory's own code generation.

## Translation

Follow `thinkwise_software_factory_translation_objects` for the general mechanics. Task-specific
points:

- A task's own label is a `transl_object` of the task's `type_of_object`, keyed by its bare
  `task_id` — confirm the live integer value rather than trusting a cached one (the enum spans
  ~150 model concepts and grows across platform versions).
- **Task parameters translate separately** — each `task_parmtr` is its own translation object,
  carrying only `transl` and `transl_form` (confirmed live: no grid/card-list/plural text, since a
  parameter never appears in a grid).
- **Confirm/Cancel buttons can have their own translation**, independent of the task's main label —
  gated by `confirm_button_has_alt_transl`/`cancel_button_has_alt_transl` on both `task` and
  `task_variant`. Leave these off unless a variant genuinely needs different button wording than
  the base task.
- `task_create_task_variant`'s `generate_transl_object` flag should stay on so the variant gets its
  own independent, editable label instead of silently inheriting the parent task's.
- `task_ref` is **not** independently translated — `ref_description` is a plain developer-facing
  description field, not user-facing text.
- **`tab_task_grp` (a task group's own label) is translatable too** — `tab_task_grp_description`
  auto-generates a bracket-placeholder `transl_object_transl` row on creation, translated the same way
  as any other object. Confirmed live `type_of_object = 16` for `tab_task_grp` — re-verify rather than
  trusting this across connectors/versions, per the usual caution on this enum.
- To find task text nobody has translated, look for the bracketed placeholder the platform
  auto-fills (`[task_id]`, `[task_parmtr_id]`), not a blank/null check.
- **Before calling a new task done, run the translation completeness gate from
  `thinkwise_datamodeling_guidelines`'s "Translating new objects" section** — a task's own label and
  every one of its parameters are separate translation objects (see above), and it's easy to translate
  the task and forget its parameters, or vice versa, without a final mechanical check.

## Pre-flight checklist

- **Plan first, per "Plan first" above**: task type, bound/unbound, parameter list, look-ups, and
  which of the four form mechanisms will be used — confirmed with the user before the first
  `stage_resource`/`stage_task` call.
- **Create in order**: `task` → `tab_task` (if table task) → `task_parmtr` → generate code group →
  assign template → generate object code. Each step's API rejects the ones that skip ahead.
- **Add the table's PK as an explicit `task_parmtr`** on any table task whose logic needs to
  identify the row — it is never supplied automatically.
- **Check `task_type_id` before assigning a template.** Only `STORED_PROCEDURE` goes through the
  control-procedure Assigning flow.
- **Both enablement gates** — task-level `use_layouts`/`use_defaults` *and* parameter-level
  input/output flags — before assuming a Default/Layout template actually runs.
- **One `tab_task` row per (table, task).** Two variants on the same table screen need two table
  variants (via `tab_variant_task_overview`), not two `tab_task` rows.
- **Propose `sub_menu = true` and `custom_display_type = icon_text_text_only_icon_only_overflow`
  (0) for a new `tab_task_grp` when confirming the plan** (per `thinkwise_software_factory_mcp_base`'s
  "Ask, don't default" convention) — both leave their platform default in place (`false`/unset) if
  not set explicitly, so call them out rather than assuming them.
- **Hiding a task isn't securing it.** An unassigned task is still callable via the API; use
  role/rights to actually restrict it.
- **Set `popup_for_each_row` deliberately on a bulk-capable task** — off (once for the whole
  selection) unless a genuinely per-row decision is required; see "Task-level row & behavior
  settings" above.
- **Bulk-copy (`task_copy_task`/`task_copy_tab`) is a starting point, not a final state** — prune
  what the target doesn't need.
- **Set a tooltip on a new task as part of finishing it** — a Software Factory validation flags a
  task, report, or prefilter left with no tooltip text at all.
- **Set `task.icon_id` to a suitable icon** as part of creating the task, per
  `thinkwise_software_factory_icons` — don't leave it unset by default. Only set a `tab_task.icon`
  override when this table's context genuinely needs a different icon than the task's own.
- **Only add `task_conditional_layout` where a parameter genuinely warrants it, and confirm the
  candidate(s) with the user before creating anything** — don't default to adding one just because a
  task has parameters, and don't finalize a plan that includes one without that confirmation. See
  `thinkwise_software_factory_conditional_layouts` for the mechanics.
- **Generating code is two separate tasks** (`task_generate_code_grp` then
  `task_add_job_to_generate_object_code`) — don't treat a successful placeholder-creation call as
  proof that code was generated.
- Confirm each task-adjacent entity's actual domain key live rather than assuming `sf/manage_tasks`
  (or `manage_datamodel`) holds on every connector — some entities (`tab_task`, `task_parmtr`,
  `tab_task_grp`) are reachable from both; the master `task`/`task_variant`/`task_ref` are not.
