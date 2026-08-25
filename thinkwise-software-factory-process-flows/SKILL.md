---
name: thinkwise-software-factory-process-flows
description: Reference guide for creating and configuring process flows (and system flows) in a Thinkwise Software Factory model — process_flow/process_action/process_step/process_variable/process_flow_schedule entities, every process action type, which types can be a starting point, control-procedure-backed logic, loops, alignment, and naming conventions. Use whenever an MCP connector with Software Factory access (e.g. sf_mcp, indicium) creates, inspects, or troubleshoots a process flow — before calling get_entity_definition/execute_odata_query/stage_resource/stage_task against process_flow, process_action, process_step, process_variable, process_flow_schedule, or process_action_start_object_available.
---

# Creating Process Flows in the Thinkwise Software Factory

A process flow is Thinkwise's mechanism for chaining tasks, reports, connectors, and decisions into
one deterministic sequence — either interactive (a user is walked through it) or, once every action
in it is non-interactive, a **system flow** that runs headless on a schedule or via API. This skill
covers the full lifecycle: the entity map, exact field-level creation steps verified against a live
model, which action types are legal starting points, naming rules, control-procedure-backed logic,
loops, layout, and translation — everything needed to build one through an MCP connector without
guessing.

Follow the connector's standard discovery→act flow; never guess entity/field/enum names — confirm
them via `get_entity_definition` first. Domain key observed in this environment:
**`sf/manage_process_flows`** — try it directly first, and only escalate to
`search_capabilities`/`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style
rejection rather than re-discovering a domain that already resolved earlier this session.

**Calling an external HTTP/REST API is its own skill, not covered end-to-end here**: build the
connection and endpoint(s) via `thinkwise_software_factory_web_connections` (a different domain,
`sf/manage_webconnections`) first, then come back here to add the `web_connection`-type action that
calls it — see the action-type table below.

## When to use a process flow at all

Use one when the application must coordinate multiple distinct actions and their order or outcome is
meaningful — a guided user sequence, a post-action continuation, an approval/confirmation flow, an
external integration pipeline, a file pipeline, a background queue worker, a scheduled
sync/cleanup, or a reusable subflow. Don't reach for one merely because several statements happen in
sequence — that's what a task's, subroutine's, or Default/Layout/Context control procedure's own logic
already covers; see `references/process_flow_design_guide.md`'s "When to use a process flow" / "When
not to" sections before modeling the flow shell, plus the recurring user-flow and system-flow patterns,
process-procedure and process-variable design principles, subflow/transaction/error-handling/scheduling
guidance, and a final review checklist — everything below this point is the API mechanics for
whichever shape you land on.

## Plan first

Before staging anything, outline the flow in plain language and get the user's explicit
confirmation — the same propose→confirm→build discipline `thinkwise_software_factory_unit_tests`
and `thinkwise_software_factory_build_planner` already use for their own artifacts, and a concrete
instance of `thinkwise_software_factory_mcp_base`'s "Shared conventions" confirm-before-mutate rule
rather than a separate process invented here. Don't start "Build order" below until this step is
done.

State, in plain language — no entity names, no SQL:

- **The trigger** — what launches the flow (a task/table/report action, a schedule, a deep link, an
  API call), and which eligible-type action will sit right after `start` (see "Starting points"
  below).
- **The action sequence** — the ordered list of real steps (tasks, reports, connectors, decision
  points) and what each one is for.
- **Branch conditions** — every point where the flow diverges (Success/Not-successful/Always, a
  message-option choice, a process-procedure route) and what decides each path.
- **Loop and error handling** — any repeated section, its exit condition and iteration cap, and what
  happens on failure at each step (retry, quarantine, visible failure) — see "Branching,
  parallelism, and loops" below and the design guide's "Error handling" section for the vocabulary.

Naming (see "Naming" below) and the loop exit condition/cap (see "Branching, parallelism, and
loops" below) are two spots this often surfaces a genuine unknown rather than an obvious answer —
raise those as open questions in the outline instead of silently picking something to keep moving.

Only once the user has explicitly confirmed the outline should proceed to "Build order" and start
staging `process_flow`/`process_action` rows. If something changes mid-build — a new branch, a
different trigger — revise the outline and re-confirm rather than patching around it silently.

## Entity map

| Entity | Role | Key |
|---|---|---|
| `process_flow` | The flow itself — name, description, trigger settings | `model_id, branch_id, process_flow_id` |
| `process_action` | One node/step in the flow, typed by `process_action_type` | `model_id, branch_id, process_flow_id, process_action_id` |
| `process_step` | One connector between two actions, with its branch condition | `model_id, branch_id, process_flow_id, process_step_id` |
| `process_variable` | Flow-scoped data passed between actions | `model_id, branch_id, process_flow_id, process_variable_id` |
| `process_flow_schedule` | A recurring trigger (system flows only, practically) | `model_id, branch_id, process_flow_id, schedule_id` |
| `process_action_start_object_available` | Which task/table/report can launch the flow at a given action | `model_id, branch_id, process_flow_id, process_action_id, process_action_start_object_id` |
| `role_process_flow_overview` | Per-role security, for the whole flow or one action | — |
| `process_flow_tag` | Free-form tags | — |

`process_action`, `process_step`, and `process_flow_schedule` are **weak entities** — stage them with
`parent_entity_set: "process_flow"` and `parent_key: {model_id, branch_id, process_flow_id}`.
`process_variable` stages flat (no parent needed — its detail nav target isn't independently
addressable through this connector, so pass the full key directly).
`process_action_start_object_available` **cannot be written through this API at all**, but this
doesn't block anything in practice — see "Starting points" below for why.

**There is no dedicated "create process flow" bound task** — `process_flow` is added directly via
`stage_resource`, unlike some other object types that need a bootstrap task first.

## Build order

Actions must exist before steps can reference them; everything else has no hard ordering beyond its
own parent.

1. `process_flow` (the flow shell)
2. `process_action` rows (every node, including `start` and `stop` — if the flow should be launchable
   by a task/table/report, include that as its own eligible-type action wired directly to `start`,
   not as something wired up separately afterward — see "Starting points" below)
3. `process_step` rows (the connectors between the actions from step 2)
4. `process_variable` rows (if the flow needs to pass data between actions)
5. `process_flow_schedule` (only if this will run unattended — see [[#Schedules]])
6. Translation of anything user-facing (see "Translation" below)

## Creating `process_flow`

Minimal valid row: `{model_id, branch_id, process_flow_id}` — everything else defaults.

- Mandatory: `model_id`, `branch_id`, `process_flow_id`, plus these which are pre-filled with sane
  defaults so you rarely need to touch them: `use_starting_points` (default `true`),
  `use_api_trigger` (default `true`, hidden field), `custom_protocol` (default `false`),
  `deep_link_allowed` (default `false`), `multiple_running_instances_allowed` (hidden, default
  `false`), `iam_custom_schedule_allowed` (hidden, default `false`).
- Optional: `process_flow_description`, `alias_process_flow_id` (hidden), `custom_protocol_alias`
  (hidden).
- **`process_flow_platform` is read-only** — auto-derived, not settable at creation.
- **`is_system_flow` is hidden and never set directly** — it's computed: the flow becomes a system
  flow automatically the instant every `process_action` in it is a non-interactive type (see the
  `system_flow_action` flag in `references/action_types.md`). Don't try to patch this field — add a
  non-interactive action instead if the flow needs to qualify.

## Creating `process_action`

Always mandatory, regardless of type: `model_id`, `branch_id`, `process_flow_id`, `process_action_id`,
`process_action_type`, `use_processes`, `confirm_start`, `auto_confirm`, `process_action_mand`.

`x_coordinate` / `y_coordinate` / `width` / `height` are editable but **never mandatory** — safe to
omit; set them later for layout (see "Alignment" below).

Per-type FK requirements (confirmed by patching `process_action_type` and re-reading field states —
check `references/action_types.md` for the full ~100-value catalog before assuming a type's exact
name):

| `process_action_type` | Extra mandatory fields | Notes |
|---|---|---|
| `start` (98) | none | Pure marker; `use_processes` itself becomes hidden for this type |
| `stop` (99) | none | Pure marker |
| `execute_task` (60) | `task_id` (mandatory) | `task_variant_id` optional |
| `execute_tab_task` (6) | `tab_id` (mandatory), `tab_task_id` (mandatory) | `tab_variant_id` stays hidden/optional until `tab_id` is set |
| `activate_detail` (1) | `tab_id` (mandatory), `ref_id` (mandatory) | **`tab_id` must be the detail/target table of the reference** (the ref's `target_tab_id`), not the source/context table — setting `tab_id` to the source table makes `ref_id` reject writes. Does **not** open as a popup/modal; it activates a detail inline in the current screen tree — see `open_document` below for a modal |
| `open_document` (2) | `tab_id` (mandatory) | `tab_variant_id` optional; `ref_id` hidden/unused. The actual modal-popup mechanism — see "Wiring runtime values through process variables" below for the fixed-input parameter that makes it open floating/modal |
| `change_filter` (330) | `tab_id` (mandatory) | The filter condition itself is **not** a plain field — see "Wiring runtime values through process variables" below |
| `web_connection` (602) | `web_connection_id` (mandatory), `web_connection_endpoint_id` (mandatory) | **The preferred choice for any HTTP/REST call — build the connection and its endpoint(s) first**, via `thinkwise_software_factory_web_connections` (domain `sf/manage_webconnections`, a *different* domain from this one). An earlier version of this note, based on inspecting `web_connection`/`web_connection_endpoint` only through `sf/manage_process_flows` (where a process action merely references a connection/endpoint by id), wrongly concluded the objects exposed "almost no writable configuration" — read through the correct domain, they carry full configuration (base URL, auth, method, path, body, headers, output parsing) and are fully buildable through the API. Once the connection exists, this action's own inputs/outputs are the pre-seeded `process_action_web_connection_endpoint_parmtr_input_parmtr` / `process_action_web_connection_parmtr_input_parmtr` (in) and `process_action_modeler_web_connection_endpoint_output` (out) rows — see that skill for the full field/enum reference and worked examples. The domain also exposes a bound migration task on this action, `task_enrichment_conv_http_connector_to_web_connection`, for converting an existing `http_connector` action. |
| `http_connector` (600) | none at the top level | **Fallback only** — reach for this over `web_connection` only for a genuine one-off call that will never be reused, or a documented gotcha a web connection hits (e.g. a reported multipart-form-on-GET issue — see `thinkwise_software_factory_web_connections`). Its configuration is not a separate child entity graph — it lives in the same `process_action_modeler_fixed_input`/`process_action_modeler_fixed_output` mechanism used elsewhere (see "Wiring runtime values through process variables" below), with `input_parmtr_id`/`output_parmtr_id` values like `http_con_url`, `http_con_http_method`, `http_con_content` pre-seeded and ready to edit. It has no reuse, no per-environment override, and no built-in response parsing — all reasons to default to `web_connection` instead. |
| `decision` (100) | none | See "Decision as a code-only step" below — this is also how you add a step that purely runs logic |

**Gotcha:** this is a `process_action`-specific instance of the general "a multi-field write can
silently drop one field, with no error" behavior documented in `thinkwise_datamodeling_guidelines`'s
quirks section — patching several fields on a fresh `process_action` in one call — especially the
composite key fields (`model_id`/`branch_id`/`process_flow_id`) alongside `process_action_id`, or
`process_action_id` alongside `process_action_type` — can leave a later field in that same call
silently reset to `null` in the response. **Separately, and not limited to batched calls**: patching
`process_action_type` or (for table-bound types) `tab_id` **or `task_id`** on its own can **overwrite
`process_action_id` with an auto-suggested id** derived from the type/table/task — e.g. setting
`tab_id` on an `activate_detail` action silently renamed it to `activate_detail_employee_absence`,
and setting `task_id` on an `execute_task` action equally renames it to `execute_task_<task_id>` —
discarding whatever id had been set moments before. **Order `process_action_id` last among the properties you set** — after `process_action_type` and
`tab_id`/`task_id` — and this is fixable in **one** combined `stage_resource`/`patch_resource` call, not
several: verified live (`RK_SCHEDULER_TEST`), staging a fresh `execute_task` action with
`process_action_type`, `task_id`, `process_action_id`, `process_action_description`, and four more fields
all in one call, in that order, committed with every field intact — `process_action_id` was not
overwritten and the description did not revert. Re-check the returned `fields` block on that one call
before moving on, rather than assuming it landed — but there's no need to split the write into several
isolated single-field calls first; try the ordered combined call and verify its result before falling
back to isolation.

`process_action_type` is an `Edm.Int32` enum keyed by numeric values (`start`=98, `execute_task`=60,
…) — patching it with the string label (`"start"`) can be rejected as an invalid value; pass the raw
integer instead (forcing a data-value interpretation on the connector if it supports one).

## Creating `process_step`

Mandatory: `model_id`, `branch_id`, `process_flow_id`, `process_step_id`, `last_process_action_id`,
`next_process_action_id`, `last_process_action_successful` (defaults to `always`=2),
`process_order_input` (default `true`), `process_order_output` (default `true`).

- `last_process_action_id` / `next_process_action_id` are plain string FKs to `process_action_id` —
  no lookup/resolve step, just pass the exact id string. **The referenced `process_action` rows must
  already exist.**
- `last_process_action_successful` enum: `not_successful` = 0, `successful` = 1, `always` = 2 — this
  is the green/red/blue branch condition from the designer canvas.
- `order_no` is **not** mandatory (defaults to `50`); `abs_order_no` is read-only and auto-computed
  — don't try to set it.
- A single action can have several outgoing `process_step` rows; per §"Branching and loops" below,
  multiple outgoing steps run in parallel, and any step's `next_process_action_id` can point at an
  action *earlier* in the flow to form a loop.

**An error message emitted during a task does not necessarily mark the process action unsuccessful —
test the actual action status, not the presence of a message.** If routing depends on a real business
outcome, return/map an explicit status variable, or confirm the task's own abort semantics genuinely
produce the unsuccessful state the flow is routing on. See
`references/process_flow_design_guide.md`'s "Branching" section for the fuller reasoning on
Success/Not-successful/Always usage.

**Inserting an action into an existing chain is manual, not automatic.** Adding a new `process_action`
between two already-connected ones doesn't re-splice the existing `process_step` for you — explicitly
edit the existing step's `next_process_action_id` to point at the new action, then add a new step from
the new action to whatever the old step used to point at.

**Renaming a `process_action` does cascade automatically**, though: the rename operation updates every
`process_step.last_process_action_id`/`next_process_action_id` that referenced the old id, and the
step's own generated id, with no manual follow-up needed.

## Creating `process_variable`

Mandatory: `model_id`, `branch_id`, `process_flow_id`, `process_variable_id`, `dom_id` (a real domain
— required, no default), `type_of_default_value` (default `constant_value`=0; other value is
`expression`=1), `available_in_deep_link` (default `false`), `mand_in_deep_link` (default `false`),
`process_input` (default `true`), `process_output` (default `true`), `sub_flow_input` (default
`false`), `sub_flow_output` (default `false`).

Optional: `default_value` (used when `type_of_default_value=constant_value`), `default_value_query`
(hidden unless `type_of_default_value=expression`), `process_variable_description`.

For API-triggered system flows, `process_property` binds a variable directly to the HTTP context
instead of a normal default: `request_method`(0), `request_path`(1), `request_query_string`(2),
`request_headers`(3), `request_body`(4), `response_code`(5), `response_headers`(6),
`response_body`(7).

## Wiring runtime values through process variables

An action's actual inputs/outputs are junction entities named
`process_action_modeler_<kind>_<input|output>` — confirmed live: `process_action_modeler_fixed_input`/
`process_action_modeler_fixed_output` (literal/enum config values and their captured results, e.g. a
connector's URL in, its response body out, or `open_document`'s floating/modal switch),
`process_action_modeler_col_input`/`_col_output` (a table column, e.g. a `change_filter` condition or
a captured row value), `process_action_modeler_task_parmtr_input`/`_task_parmtr_output` (a task's own
parameters), plus `sub_flow`/`report_parmtr`/`message_broker_message` variants for those action
types — pick the one matching what the action actually is, not one generic table for all of them.
**For a `web_connection` action specifically**, the equivalents are
`process_action_web_connection_endpoint_parmtr_input_parmtr` (endpoint input),
`process_action_web_connection_parmtr_input_parmtr` (connection-level input), and
`process_action_modeler_web_connection_endpoint_output` (output) — see
`thinkwise_software_factory_web_connections` for the full field reference.

**A generically-named `process_action_output_parmtr` entity also exists and looks like "the" output
entity from its own definition (it even carries the same `output_parmtr_id` enum as
`process_action_modeler_fixed_output`) — but `add` against it is rejected outright (403) through this
API, confirmed live.** It appears to be a newer, unified/read-oriented scheme that isn't (yet) a valid
write path here. For capturing a connector-type action's output (`http_connector`, `ftp_connector`,
`db_connector`, etc.) into a variable, use `process_action_modeler_fixed_output` instead — same
pre-seeded/edit-only pattern as `process_action_modeler_fixed_input` below, keyed by `output_parmtr_id`
(e.g. `http_con_content` for an HTTP response body), with a `process_variable_id` field to set.

**These rows are pre-seeded, not freely addable.** The moment a `process_action`'s type (and, for
column-based ones, its `tab_id`) is set, one placeholder row per relevant column/parameter already
exists — e.g. one `_col_input` row per column of the action's table, one `_fixed_input` row per input
parameter that type supports. **Staging an `add` on any of these child entities is rejected outright**
(before you even get to set a field) — query for the existing placeholder row first (filter by
`process_action_id`), then stage an `edit` against its full composite key (including
`tab_id`/`col_id`/`input_parmtr_id`/`task_parmtr_id` as applicable) instead.

**Worked pattern — filtering a popup to the row that triggered the flow** (the shape behind
"double-click a row → modal popup of related, filtered records"):

1. The trigger is normally an `execute_tab_task` action running a table task bound via
   `grid_double_click`. **The row's data does not reach the flow through
   `process_action_modeler_col_output` on this action** — that entity stages and commits without error
   but silently captures nothing usable for an `execute_tab_task`. Instead, add a `task_parmtr` on the
   underlying table task itself (e.g. `employee_id`) with **both** `task_input=true` (so a
   `tab_task_parmtr` binding can auto-fill it from the selected row's column) **and**
   `task_output=true` (so its value becomes readable by the flow).
2. Create a `process_variable` with a matching `dom_id`.
3. On the triggering `execute_tab_task` action, edit its pre-seeded
   `process_action_modeler_task_parmtr_output` row for that `task_parmtr_id`, setting
   `process_variable_id` to the variable — this is what actually carries the selected row's value into
   the flow.
4. Open the popup with an `open_document` action, then follow it with a `change_filter` action on the
   same table. Edit that action's pre-seeded `process_action_modeler_col_input` row for the FK column,
   set `assignment_method='variable'` (`process_variable_id` often auto-resolves on an unambiguous
   name/domain match, but verify it).
5. For the popup's modal behavior itself: `open_document` actions get a pre-seeded
   `process_action_modeler_fixed_input` row keyed by `input_parmtr_id='open_doc_floating'` — edit it to
   `assignment_method='literal_constant'`, `constant_enum_value='open_doc_floating_modal'`.

**A given `process_variable` can only be the output target of one binding per action.** Assigning two
different output rows on the same action to the same variable (e.g. both a `col_output` and a
`task_parmtr_output` pointed at the same variable, left over from an earlier wrong attempt) causes a
real duplicate-key failure on the second commit — clear/null the first binding before setting the
second.

## Schedules

Weak entity under `process_flow`. Mandatory: `model_id`, `branch_id`, `process_flow_id`,
`schedule_id`, `recurrence_type` (default `daily`=0; other values `weekly`=1, `monthly`=2),
`recurrence_day` (default `1`), `occur_type` (default `once`=0; other values `recurring`=1,
`recurring_all_day`=2), `occur_once` (mandatory **only** when `occur_type=once` — a `TimeOfDay`).

Day-of-week flags (`monday`…`sunday`) are hidden and only relevant when `recurrence_type=weekly`;
month-position fields only matter when `recurrence_type=monthly`. Minimal daily-at-a-fixed-time
schedule: `{model_id, branch_id, process_flow_id, schedule_id, occur_once: "02:00:00"}` with
everything else left at its default.

Schedules only make practical sense once the flow qualifies as a system flow (every action
non-interactive) — a schedule on a flow that still has an interactive action has nothing meaningful
to fire unattended.

## Starting points — what can launch a flow, and a hard write-API limitation

`process_action_start_object_available` records which real object (a table row, a report, a task) can
launch the flow, landing execution at a specific `process_action` rather than always at the literal
`start` node. It references `process_action_id` directly; which kind of object is the candidate is
expressed by *which* of its FK-shaped columns is populated (`tab_id`/`tab_variant_id`,
`report_id`/`report_variant_id`, `task_id`/`task_variant_id`) — there's no separate discriminator
column.

**Only eight `process_action_type` values are ever observed as starting points**, confirmed against
real data across ten live models:

| Type | Why it can start a flow |
|---|---|
| `activate_detail` (1) | Opening/activating a detail is itself a user launch point |
| `open_document` (2) | Opening a document can be the trigger |
| `add_record` (3) | Creating a new row can launch a flow (e.g. a wizard) |
| `edit_record` (4) | Editing a row can launch a flow |
| `delete_record` (5) | Deleting a row can launch a flow |
| `execute_tab_task` (6) | The overwhelming majority of real starting points — any table task |
| `execute_task` (60) | A standalone task launching a flow |
| `execute_system_task` (61) | A system task launching a (system) flow |

Every other type — including `start`, `stop`, `execute_tab_report`/`execute_report`/`generate_report`,
`activate_grid`/`activate_form`, `decision`, and every connector type — was **never** observed as a
starting point. `start`/`stop` are excluded because they're the flow's own implicit entry/exit, not
things a user or API launches into; reports have schema support (`report_id`/`report_variant_id`
columns exist) but weren't exercised in any live model sampled — treat report-triggered starts as
theoretically schema-supported but unverified in practice.

**`process_action_start_object_available` itself is read-only through this MCP write API** — both
`add` and `edit` via `stage_resource` return `403`/`staging_request_failed`, even against an existing
row, and the only bound write operation found, `task_reset_process_action_start_object` (zero
parameters), *clears* a starting point rather than selecting one. **This is not actually a blocker,
and no manual step in the Software Factory's own UI is needed** (correcting an earlier version of this skill that claimed
otherwise): **the trigger object simply needs to be modeled as its own eligible-type action wired
directly to `start` inside the flow.** Any action of one of the eight types above that sits
immediately after `start` is automatically usable as a launch point for its underlying task/table/
report — `process_action_start_object_available` reflects that structural fact rather than being a
separate switch you flip. Confirmed live (2026-07-28): a menu-triggered flow needs an `execute_task`
action (`task_id` = the menu's trigger task) as the first action after `start`; putting the trigger
task outside the flow and hoping to "wire it up" separately (via the Software Factory's own UI or
otherwise) is the mistake to avoid — build it into the flow's own action chain instead.

## Process action types

The full ~100-value catalog, grouped by category with each type's `system_flow_action` eligibility,
is in `references/action_types.md` — read it before picking a type for a new action, rather than
guessing a name. Roughly 57 of the ~100 types are legal inside a system flow (essentially every
connector, `decision`, `execute_system_task`/`execute_system_sub_flow`, AI/ML, IAM, file, and
`start`/`stop`); UI-bound types (`activate_form`, `add_record`, `edit_record`, `execute_tab_task`,
filter/sort/grid-navigation, etc.) are not, since a system flow runs with no session to act on.

**There is no dedicated "control procedure" action type.** Control-procedure logic never gets chosen
directly on the canvas — see "Process logic" below for exactly how it attaches.

### Decision as a code-only step

`decision` normally routes execution based on a condition, but it's also the right action to reach
for whenever a step's *only* job is running code — with `use_processes` checked and a single outgoing
`process_step` set to `always` (no real branching), it's a clean way to run a control procedure
(sum a total into a process variable, validate something, transform data) between two other actions
without any task, report, or connector attached to it at all.

Example: a `decision` action `calculate_totals`, `use_processes=true`, one outgoing step at
`last_process_action_successful=always` to the next action — its assigned control procedure sums
order lines into a process variable that a later `execute_task` action then consumes via
`{order_total}` substitution.

This is one specific use of a process procedure. For the fuller decision — when a process procedure is
warranted at all vs. when it's the wrong place for the logic, and the best-practice checklist for
writing one — see `references/process_flow_design_guide.md`'s "When to use a process procedure"
section.

### Show message (`show_msg`) — presenting a message and branching on the choice

`process_action_type = show_msg` (enum value `350`) presents a modeled message (set `msg_id` on the
action) and, for a choice message, lets the flow branch on which `msg_option` the user picked. Full
message-modeling mechanics (severity, location, options, database capture, calling from SQL) live in
`thinkwise_software_factory_messages` — load that skill before creating or editing the `msg`/`msg_option`
rows themselves; this section covers only how the action wires into the flow.

**The ordinary `last_process_action_successful` step condition (`not_successful`/`successful`/`always`)
is too coarse the moment a message has more than one affirmative or more than one negative option** —
it can't distinguish, say, `yes` (`status_code=0`) from `yes_always` (`status_code=1`). To route on the
exact chosen option, capture its numeric status code into a process variable and branch on that instead:
the `show_msg` action has a pre-seeded `process_action_modeler_fixed_output` row keyed
`output_parmtr_id='status_code'` (the same generic status-code output every action type exposes, not
`msg`-specific) — following the usual pre-seeded/edit-only pattern above, query for that existing row
filtered by `process_action_id`, edit it to set `process_variable_id`, then follow with `decision`
action(s) testing that variable against each option's `msg_option_status_code`. For a plain single
yes/no choice, the ordinary `successful`/`not_successful` step condition on the two outgoing
`process_step`s is enough and the status-code capture is unnecessary.

## Branching, parallelism, and loops

Multiple outgoing `process_step` rows from one action fan out and run **in parallel**; a later action
that several branches converge into only fires once every parallel branch feeding it has completed.
Only one flow instance is active per user at a time.

**A loop is just a `process_step` whose `next_process_action_id` points at an action earlier in the
same flow** — nothing in the schema distinguishes a "loop" connector from any other; it's drawn (or
staged) exactly the same way, just aimed backwards. The standard shape is two pieces: the action(s)
doing repeated work, and a `decision` that tests a process variable each pass and either loops back or
lets execution continue.

Example — paging through an HTTP API until there's no more data: variables `page_number` (int,
default `1`), `has_more_pages` (bool), `iteration_count` (int, default `0`); a `web_connection` action
`get_page` (endpoint path/query string uses `{page_number}` — see
`thinkwise_software_factory_web_connections`); an `execute_task` action
`store_page_results` that saves the page and increments both `page_number` and `iteration_count`
(via its control procedure); a `decision` action `check_more_pages` whose `successful` step loops back
to `get_page` when `has_more_pages=true`, and whose `not_successful` step continues to `stop`.

> **Warning:** nothing in the model stops a loop from running forever — there's no built-in
> step-count fuse, and "one flow instance per user" is no protection for a system flow: a scheduled
> flow that loops without ever satisfying its exit condition keeps its Indicium worker busy
> indefinitely, and a recurring schedule can stack further runs on top if
> `multiple_running_instances_allowed` is on. **Prevention strategy:** never let the loop depend on
> the business condition alone — add an unconditional counter variable incremented every pass, AND
> the real exit condition with a hard `iteration_count < max_iterations` cap, treat hitting the cap
> as a distinct alertable outcome (set an output variable and route it to a notification/log action)
> rather than silent truncation, and prove the exit condition with a small cap in the Process flow
> monitor before raising it to a realistic ceiling.

**The exit condition and the iteration cap are business decisions, not implementation defaults** —
if the user hasn't said what business condition should end the loop or what `max_iterations`
ceiling is safe, ask rather than picking a number unilaterally, per
`thinkwise_software_factory_mcp_base`'s "Shared conventions" ask-don't-default rule. The "prove the
exit condition with a small cap" technique above is a testing step for validating a cap/condition
the user has already settled on — it doesn't substitute for asking what that cap/condition should
be in the first place.

## Every path must end at `stop`

Check every branch — including message/error branches and loop exits — actually reaches a `stop`
action, not just the main line. A path that dead-ends on a non-stop action leaves that flow instance
incomplete. Treat this as a mandatory check before considering any flow done, the same way you'd check
every `process_step` you added actually has a valid `next_process_action_id`.

## Process logic and control procedures

Two distinct mechanisms carry real SQL logic in a process flow, and confusing them is the most common
mistake:

1. **The action's own `use_processes` flag** ("Use process procedure") opts that specific
   `process_action` into the `PROCESSES` code group — a control procedure assigned here runs as part
   of the action itself (see "Decision as a code-only step" above for the most common use of this).
2. **The far more common pattern**: an `execute_task`/`execute_system_task` action simply calls a task
   whose own logic type is **Stored procedure**, and the *task's* control procedure (in the `TASKS`
   code group) is where the real work happens — parsing a connector's response, writing rows, etc.

Either way, creating and assigning that control procedure follows the full lifecycle documented in
`thinkwise_software_factory_create_control_procedures` — load that skill before writing the actual
SQL: check `branch_rdbms_type` first, scaffold with **Generate code group** before hand-guessing the
business-logic variables available to a `PROCESSES`/`TASKS` code-group procedure, then assign and
verify by reading the generated code, not just trusting a "successful" status.

## Alignment

Layout lives entirely on `process_action`: `x_coordinate`, `y_coordinate`, `width`, `height` — plain
numbers, no separate design table, no persisted grid-snap metadata (snapping is designer-UI behavior
only). A convention taken from a real flow's actual coordinates: keep `width`/`height` constant across
every action, run the main "happy path" along one horizontal lane (`y_coordinate` unchanged), and step
`x_coordinate` by a fixed increment (~48–56 units) per action. Give a `decision`'s branches their own
row above/below the main lane, then reconverge them back onto the main `y` before `stop`. Since none of
this is mandatory, it's safe to omit at creation time and set once the flow's logic is proven — but do
set it before calling the flow finished, since an unlaid-out canvas is hard for the next person (or
agent) to read.

## Naming

**A process flow's `process_flow_id` must never reuse an existing task's `task_id` or a report's
`report_id`/`tab_report_id` in the same model.** Nothing in the write API actually rejects this
collision — staging a `process_flow` with an id identical to an existing task's id was accepted
without a validation warning — which is exactly why this has to be enforced by discipline rather than
relied on as a platform guarantee: query existing `task` and `report`/`tab_report` ids in the target
model first and confirm no overlap before picking a `process_flow_id`.

**Before naming a new flow, look at the other process flows already in the model** and match
whatever local convention they establish — conventions observed vary genuinely by model/team, so
there is no single universal answer:

- **Plain business flows** (the common case): snake_case, no fixed prefix, named for the outcome or
  trigger rather than a strict word order — `sales_invoice_approve_print`,
  `calculate_and_store_route`, `customer_create_contact_person`, `deep_link_to_sales_invoice`. Pick
  whichever ordering (object_verb vs. verb_object vs. outcome-first) the sibling flows in this model
  already use, rather than introducing a new one.
- **System/background flows**, when a project wants them visually distinguishable from interactive
  ones, commonly take a **`system_flow_`** prefix (`system_flow_clean_up`,
  `system_flow_automatic_thinkstore_refresh`) — Thinkwise's own built-in framework flow
  (`tsf_system_flow_run_tsf_optimize`) uses `tsf_system_flow_` for the same reason. This is a chosen
  convention, not an enforced one: plenty of real system flows (e.g. `save_customer_coordinates`,
  `check_uta_vm_status`) have no such prefix at all — `is_system_flow` is a computed property, never
  something the name has to signal.
- Some teams use a lighter **`flow_`** or **`pf_`** micro-prefix instead, especially for many small,
  similar flows following one template (`flow_copy_<x>_to_new_record` repeated per table).
- All observed names are lowercase snake_case — no camelCase, no spaces, ever.

- **If the model has no existing process flows to pattern-match against and no other convention is
  obvious, don't default silently** — per `thinkwise_software_factory_mcp_base`'s "Shared
  conventions" (ask, don't default), propose plain outcome-descriptive snake_case with no prefix as
  your recommendation (matching general Thinkwise naming guidelines — see
  `thinkwise_datamodeling_guidelines`, purpose over plumbing, same principle as control procedure
  naming) and get the user to confirm or correct it before committing to a `process_flow_id`.

## Translation

The flow's own description, its actions' descriptions, any message text, and process variable
descriptions are all translatable objects. Don't leave newly-created text in the source language only
— load `thinkwise_software_factory_translation_objects` for the `transl_object`/`transl_object_transl`
mechanics and the approval workflow before considering a new process flow finished.

## Pre-flight checklist

- Confirm the domain key via `search_capabilities`/`get_available_domains` rather than assuming
  `sf/manage_process_flows` holds for this connector.
- Query existing `task`/`report` ids in the target model before picking a `process_flow_id` — no
  collision is enforced automatically.
- Look at sibling process flows already in the model and match their naming convention before
  introducing a new one.
- Create `process_action` rows before any `process_step` that references them.
- Set a fresh `process_action`'s fields in **one** combined call, ordered so `process_action_type` and
  `tab_id`/`task_id` come before `process_action_id` — verified live, this lands every field correctly in
  a single round trip. Re-read the returned `fields` block on that call before moving on.
- Pick the right per-type mandatory FK for each `process_action` (see the type table above) —
  default to `web_connection` for any HTTP/REST call (see
  `thinkwise_software_factory_web_connections` to build the connection first); reach for
  `http_connector` only as a documented fallback for a genuine one-off call or a specific gotcha.
- Don't try to patch `is_system_flow` or `process_flow_platform` directly — both are
  computed/read-only.
- **A flow's trigger object (task/table/report) must be its own eligible-type `process_action` wired
  directly to `start`** — not left outside the flow. `process_action_start_object_available` can't be
  written via this API, but that's fine: it's a structural reflection of the action chain, not a
  separate switch, so no manual step in the Software Factory's own UI is needed once the trigger action
sits right after `start`.
- Check every branch (including loops and error paths) actually reaches a `stop` action.
- A `duplicate_key`-style error on commit is **not a reliable signal either way** — it has been
  observed both as a false negative (the row had actually persisted) and as a real failure. Re-query
  the row after any duplicate-key error before trusting either outcome. **Confirmed cause of one such
  false negative**: adding the very first `process_action` (any type, not just `start`) to a
  brand-new, empty flow can silently auto-bootstrap a paired `start`+`stop` action set as a
  side effect — mirroring what the flow's own bound `task_add_start_stop_process_action` does —
  before your own insert commits, producing a spurious duplicate-key error even though both markers
  now exist correctly. Re-query `process_action` for the flow before adding `start`/`stop` yourself.
- Any loop needs an unconditional counter variable AND'd into its exit condition with a hard cap —
  never let the loop depend on the business condition alone.
- Attach control-procedure logic via `use_processes` on the action itself, or via the *target task's*
  own logic for an `execute_task`/`execute_system_task` action — then follow
  `thinkwise_software_factory_create_control_procedures` for the actual SQL lifecycle.
- Set `x_coordinate`/`y_coordinate`/`width`/`height` before calling the flow done, even though none
  are mandatory — an unlaid-out canvas is hard to read.
- Translate every new description/message via `thinkwise_software_factory_translation_objects` before
  considering the flow finished.
