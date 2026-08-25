---
name: thinkwise-software-factory-deployment
description: Reference guide for starting, monitoring, and controlling a Thinkwise Software Factory definition-generation job — generating the application definition alone, or the full generate/validate/build/deploy pipeline ("complete creation") — via the sf/software_development domain. Use whenever an MCP connector with Software Factory access starts, checks, cancels, or resolves a paused step of a generation/validation/deployment job, or before calling get_task_definition/stage_task/execute_odata_query against add_job_to_generate_definition, definition_generation_overview, definition_generation_step, or definition_validation_overview.
---

# Generating and deploying a Thinkwise Software Factory application

Definition generation and deployment are modeled as **jobs** in a dedicated domain,
`sf/software_development` — distinct from the `sf/manage_datamodel`-style domain most other
Software Factory skills use for tables/tasks/screens. Confirm connector, `model_id`, and
`branch_id` per `thinkwise_software_factory_mcp_base` before starting a job; a repository commonly
holds several models, each with several branches, and a job always runs against one specific
`(model_id, branch_id)` pair.

Canonical flow, same as every other domain: `search_domain_capabilities` (intent `action`, keywords
like "generate definition") → `get_task_definition` → `stage_task` → `commit_resource`. Do not guess
task or parameter names — this domain has several similarly-named tasks (see below) that are easy to
confuse.

## The two tiers of job

| Tier | Task | Binding | What it does |
|---|---|---|---|
| Generate definition only | `add_job_to_generate_definition` | Unbound | Runs just the definition-generation step(s). Verified live end-to-end this session. |
| Full pipeline ("complete creation") | `task_add_job_to_do_complete_creation` | Bound to `definition_generation_overview` | One job that can chain generate → validate → generate source code → write code files/program objects → execute source code → unit/smoke tests → sync to IAM or create a deployment package, each stage individually toggled on/off with its own error-handling policy. |

**A near-duplicate unbound task exists, unverified**: `add_job_to_generate_definition_checks` has an
*identical* mandatory-parameter signature to `add_job_to_generate_definition` (`model_id`,
`branch_id`, `error_handling`, `debug`) and the same "Generate definition" description. A bound
variant, `task_add_job_to_generate_definition_checks` (on `definition_generation_overview`), is
described "Execute all code files" instead — a different label for what looks like the same
parameter set. None of the `_checks`-suffixed variants were exercised this session; only the plain
`add_job_to_generate_definition` was run and confirmed to work as expected. If a `_checks` variant
seems like the better fit for a task, verify its actual behavior against a live job before relying on
it, and update this note with what's found.

### Tier 1 — generate definition only

```
stage_task(add_job_to_generate_definition, properties: [model_id, branch_id, error_handling, debug])
→ commit_resource
```

| Parameter | Type | Notes |
|---|---|---|
| `model_id` | string | mandatory |
| `branch_id` | string | mandatory |
| `error_handling` | enum (byte) | mandatory — `pause_and_await_user_input` (0, default-looking), `skip_and_continue` (1), `abort_generation` (2) |
| `debug` | boolean | mandatory |
| `job_id` | int64 | optional — leave unset for a new job |

### Tier 2 — full pipeline (`task_add_job_to_do_complete_creation`)

Bound to `definition_generation_overview`. Every stage is an independent boolean flag with its own
error-handling enum where relevant — nothing here has an obviously-safe default, so **ask the user
which stages to run** rather than turning them all on:

| Flag | Type | Governs |
|---|---|---|
| `execute_complete_creation` | enum | `complete` (1), `selected` (0), `selected_start_with_none` (2), `custom` (3) — which of the flags below apply |
| `creation_preset_id` | string, optional | a saved preset of these settings |
| `generate_definition` | bool | run definition generation |
| `generate_definition_error_handling` | enum | same 3 values as Tier 1's `error_handling` |
| `validate_definition` | bool | run validation |
| `validate_definition_error_handling` | enum | `validation_error` (0), `validation_warning` (1), `any_validation_msg` (2), `do_not_abort_creation` (3) |
| `generate_source_code` | bool | generate source code |
| `upgrade_method` | enum | `smart` (0), `full` (1) |
| `write_code_files` | bool | write code files to storage |
| `write_prog_objects` | bool | write program objects to storage |
| `execute_source_code` | bool | **actually runs the generated code against a database** |
| `runtime_configuration_id` / `runtime_host` / `runtime_db_name` | string, optional | target for `execute_source_code` |
| `execute_source_code_error_handling` | enum | `abort` (0), `ignore` (1), `await_user_input` (3) |
| `execute_unit_tests` / `execute_smoke_test` | bool | test execution |
| `run_sync` | bool | **syncs to an external IAM or produces a deployment package** |
| `sync_run_type` | enum | `sync_iam` (0), `sync_iam_to_disk` (1), `create_deployment_package` (2) |
| `sync_target_iam_id` / `sync_method` (`api`=0/`db`=1) / `sync_url` / `sync_host` / `sync_db_name` | — | sync target detail, depends on `sync_run_type` |
| `debug` | bool | |
| `job_id` | int64, optional | |

`execute_source_code` and `run_sync` are the two flags that reach outside the model repository
itself (a live database, an external IAM, or a package artifact) — treat enabling either as the kind
of action that needs explicit user confirmation, per the "hard to reverse / affects shared systems"
guidance, not just the general modeling ask-before-default rule.

A related bound task, `task_add_job_to_do_complete_creation_checks` (same entity, no mandatory
parameters beyond the bound key), was found via metadata only and not exercised — likely a dry-run
"can this job run" check, but confirm behavior live before depending on it.

## Ask, don't default

`error_handling`/`generate_definition_error_handling`, `debug`, and every stage-toggle in the
complete-creation pipeline are genuine operational decisions, not mechanical CRUD fields — per
`thinkwise_software_factory_mcp_base`'s ask-don't-default rule, ask the user rather than silently
defaulting, especially for `execute_source_code`/`run_sync`. `pause_and_await_user_input` is the
lowest-risk error-handling choice when unsure, since it stops for a decision instead of silently
skipping or aborting — but confirm even that with the user rather than assuming it.

## Verified gotcha — enum parameters need the numeric value or the exact display label, not the enum key

Passing an enum parameter's snake_case key (e.g. `"pause_and_await_user_input"`, exactly as shown in
`enumValues` by `get_task_definition`) as a `stage_task`/`patch_resource` property value **fails**
with `invalid_input` — that key is an internal identifier, not a valid input value. What works,
verified live: the numeric value as a string with `value_kind: "data"` (e.g.
`{"property": "error_handling", "value": "0", "value_kind": "data"}`), or presumably the exact
human-readable display label. After a failed enum property in a multi-property `stage_task`/
`patch_resource` call, later properties in the same ordered list may be silently skipped
(`not_applied`) — always re-read the returned `fields` and re-patch anything that didn't take, per
the mcp_base multi-property-patch hazard.

## Verified gotcha — `branch` has no `description` column

Querying `/branch?$select=model_id,branch_id,description` fails — `branch` has no `description`
property. Use `$select=model_id,branch_id` (or no `$select`) when listing branches for a model, per
the mcp_base "verify unfamiliar field names before querying" rule.

## Discovering the branch(es) for a model

```
execute_odata_query(sf/software_development, /branch?$filter=model_id eq '<MODEL_ID>'&$select=model_id,branch_id)
```

Only ask the user to pick a branch (per mcp_base) if more than one row comes back.

## Monitoring a job

`definition_generation_overview` (key `job_id, model_id, branch_id`) is the job header:

```
execute_odata_query(sf/software_development,
  /definition_generation_overview?$filter=model_id eq '<MODEL_ID>' and branch_id eq '<BRANCH_ID>'
  &$orderby=job_id desc&$top=1
  &$select=job_id,definition_generation_status_name,progress,start_date_time,finish_date_time)
```

`definition_generation_status` enum: `scheduled` (0), `executing` (1), `wait_for_user` (2),
`successful` (3), `failed` (4), `cancelled` (5), `aborted` (6), `warning` (7), `info` (8).

For step-level detail, `definition_generation_step` (key `job_id, definition_generation_step_id`)
lists each step with its own `definition_generation_step_status` (`not_executed` 0, `executing` 1,
`wait_for_user` 2, `successful` 3, `failed` 4, `warning` 5, `skip` 6), `order_no`, and the
`control_proc_id` actually being run for that step. `definition_generation_step_log` carries
per-step error detail (`error_no`, `error_severity`, `error_msg`, `error_proc`, `error_line`) — read
this when a step fails to get the actual database/generation error rather than just the status.

A parallel `definition_validation_overview` entity (key `job_id, model_id, branch_id`) exists for
validation with its own status enum (same 9 values as generation) and a
`definition_validation_log` detail; how a standalone validation job gets *started* wasn't
discovered this session (validation inside Tier 1/2 is driven by the generation/complete-creation
job itself) — treat as read/monitoring-only until verified.

## Resolving a paused step (`wait_for_user`)

When `error_handling`/`generate_definition_error_handling` is `pause_and_await_user_input` and a
step's status becomes `wait_for_user`, resolve it with a bound task on that
`definition_generation_step` row (key `job_id, definition_generation_step_id`):

- `task_abort_definition_generation` — abort the job.
- `task_ignore_and_continue_definition_generation` — skip this step, continue.
- `task_retry_definition_generation` — retry the same step.

For a standalone validation job, the equivalent is `task_abort_definition_validation` (bound to
`definition_validation_overview`, mandatory `job_id`).

## Cancelling a job

`task_cancel_job` (bound to `definition_generation_overview`, mandatory `job_id`) cancels a running
job after its current step finishes — not an instant kill. The same task exists bound to
`definition_validation_overview` for a standalone validation job.

## Pre-flight checklist

- Pin connector/model/branch first (`thinkwise_software_factory_mcp_base`); confirm the branch via
  `/branch?$select=model_id,branch_id` (no `description` field).
- **Ask which tier** the user wants — definition generation alone, or the full complete-creation
  pipeline — and ask about every non-obvious flag (error handling, debug, and every stage toggle in
  Tier 2), rather than defaulting.
- Treat `execute_source_code` and `run_sync` as high-blast-radius flags needing explicit
  confirmation, since they reach a live database or external IAM/deployment artifact.
- Pass enum parameters as their numeric `data` value (or exact display label) — never the snake_case
  enum key from `get_task_definition`'s `enumValues`.
- After staging with multiple properties, re-read the returned `fields` and re-patch anything that
  shows `not_applied` before committing.
- After commit, verify the job actually started by querying `definition_generation_overview` for the
  new `job_id` rather than trusting `{"committed": true}` alone.
- Poll `definition_generation_overview`/`definition_generation_step` to track progress; check
  `definition_generation_step_log` on any `failed` step for the real error.
- If a step hits `wait_for_user`, resolve it explicitly (abort/skip/retry) — the job will not
  progress on its own.
