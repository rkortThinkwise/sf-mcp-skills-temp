# Process flow design — when to use one, and how to shape it

This file is about *design*, not API mechanics: whether a process flow is the right tool at all, which
recurring shape fits a given need, when to reach for a process procedure or a subflow, and how to
design process variables, error handling, and scheduling. For the entity map, exact field requirements,
and verified-live quirks, see `SKILL.md` — this file assumes those mechanics and adds the reasoning
behind the choices.

## When to use a process flow

Use one when the application must coordinate multiple distinct actions and their order or outcome is
meaningful. Strong candidates:

- A guided user sequence: create → enrich → review → generate report → open result.
- A post-action continuation: save an order, then open its planning screen.
- An approval or confirmation flow with explicit alternatives.
- An external integration pipeline: prepare → authenticate → call → parse → validate → persist.
- A file pipeline: list/read → transform → import → archive or quarantine.
- A background queue worker.
- A scheduled synchronization or cleanup.
- A reusable sequence invoked as a subflow.
- An API-facing workflow with a defined request/response contract.

## When not to use a process flow

Don't reach for one merely because several statements happen in sequence — that's what a single task's
own logic already does. Prefer instead:

- A **task** for one cohesive user command.
- A **subroutine** for reusable business/database logic with inputs and outputs
  (`thinkwise_software_factory_subroutines`).
- A **Default/Layout/Context** control procedure for one logic concept
  (`thinkwise_software_factory_create_control_procedures`).
- A **Trigger/Handler/constraint** for integrity that must hold on every write path.
- A **process procedure** only for immediate route selection — not as a substitute for the whole flow
  (see "When to use a process procedure" below).
- An **event/API/message-broker trigger** instead of polling, whenever work is naturally event-driven.

A long chain of invisible automated actions with no user interaction and no real branching is often
better as one task/subroutine, unless separate connector actions, retry boundaries, monitoring, or
reuse actually justify orchestration.

## User process flow vs. system flow

| Characteristic | User process flow | System flow |
|---|---|---|
| User interaction | Yes or potentially yes | None |
| Execution context | Current user and permissions | Indicium/system context |
| Typical start | UI action, deep link, API trigger | Schedule, API, system event/manual run |
| Scheduling | No | Yes (see `SKILL.md`'s Schedules section) |
| Typical purpose | Guidance and navigation | Integration, batch, queue, maintenance |

**Don't convert an interactive flow into a system flow by assuming default answers for user choices.**
If unattended behavior is genuinely needed, model a separate, deliberately autonomous flow instead of
stripping the interactive one down.

## Common user-flow patterns

- **Guided creation** — Start task → Add record → Open record → Activate required detail → Start
  follow-up task/report. Use when each step genuinely needs UI input in a fixed order; keep optional
  exploration outside the flow — an overly rigid wizard frustrates experienced users.
- **Post-save navigation** — Add/Edit row → Go to row → Open or activate a related document. Note that
  Add/Edit-row start triggers only work with a Form component and Auto-save disabled; auto-saving grid
  navigation can conflict with subsequent flow actions.
- **Confirmation and choice** — Task result → Show message with options (Continue / Return to edit /
  Cancel). Use explicit message-option status codes and meaningful labels (see
  `thinkwise_software_factory_messages` and `SKILL.md`'s `show_msg` section); handle close/cancel and
  unexpected status. Don't reach for a process procedure when the visible choice is already clear from
  the diagram.
- **Review and approval** — Open item → user reviews → Approve/reject task → refresh/open next work
  item. The database must still enforce authorization and valid status transitions; the flow guides the
  user, it is not the integrity boundary.
- **Generate and open output** — Collect parameters → Generate report/file → Open/download result →
  optional notification. Separate generation failure from display failure — a successfully created
  document shouldn't be regenerated merely because opening it failed.

## Common system-flow patterns

- **External API pipeline** — Initialize configuration → Acquire/refresh token → Build request → Web
  connection → check HTTP status → Parse and persist, or Log/retry/quarantine. Keep credentials in
  secure configuration, not literals; treat transport success, HTTP success, valid payload, and business
  acceptance as separate checks (see "Error handling" below).
- **Incremental synchronization** — Read last successful watermark → request changes since watermark →
  validate and upsert idempotently → commit page/batch → advance the watermark only after success. Use
  overlap/deduplication when the source's timestamps are imprecise.
- **Queue worker** — Claim next open queue item atomically → mark processing with lease/attempt →
  execute work → complete, or retry/quarantine → repeat next scheduled instance. The claim must be
  atomic — "select open, then update" lets two workers take the same record; use a queue table with a
  proper atomic claim when multiple instances may run.
- **File ingestion** — List files → claim/move to processing → read and validate → import → archive on
  success → quarantine plus diagnostics on failure. Use a stable file identity/checksum to prevent
  re-import; never delete a source file before durable success is established.
- **Report and distribute** — Determine recipients/data → generate report → send email/notification →
  store delivery result. A unique delivery key prevents duplicate emails when a run is retried.
- **Periodic maintenance** — Check whether work is necessary → process a bounded batch → record
  metrics/result → stop. Examples: retention cleanup, index/statistics maintenance, stale-lock cleanup,
  cache refresh, security scans. Schedule costly maintenance during low activity and coordinate with
  other nightly jobs.

## Branching: error message is not the same as action failure

**An error message emitted during a task does not necessarily mark the process action unsuccessful.**
Test the actual action status, not the presence of a message. If routing depends on a real business
outcome, return/map an explicit status variable, or make sure the task's own abort semantics actually
produce the unsuccessful state the flow is routing on — don't assume "it showed an error" and "the
action failed" are the same thing.

Use modeled Successful/Not-successful paths for ordinary error routing; use Always only for cleanup,
final logging, or a decision that must inspect status regardless of outcome — and only when that's
genuinely safe after both outcomes. Make failure endpoints visible rather than silently stopping, and
give steps/actions outcome-oriented names.

## When to use a process procedure

Use one when Success/Not-successful is insufficient and the next path depends on several variables:

- HTTP status family plus response content.
- Order state plus user choice.
- Whether output exists, a report is requested, or changes were found.
- Retry count plus error classification.

A **Decision action** always uses a process procedure and can route through Always-type follow-up
steps — useful at the start of a system flow to initialize/autonomously load state, or at a clear
decision point needing no external/UI action (see `SKILL.md`'s "Decision as a code-only step").

Also reach for one to conditionally skip an optional follow-up (skip printing when there's no printable
result, notify only past a threshold, clean up only when a temporary artifact was created, continue
pagination while a next-page token exists) or to order several enabled immediate branches when the
order genuinely depends on runtime variables — if the order is always fixed, encode it directly in the
graph instead.

### When not to use a process procedure

- Core business mutations better owned by a task/subroutine/handler.
- Calling external services — use connector actions instead.
- Large data transformations hidden inside routing code.
- Authorization or integrity rules that must apply outside this flow.
- Routing already expressible clearly with plain Success/Not-successful connections.
- A maze of invisible conditions that makes the diagram misleading.
- Long-running work or retry waiting.

**The test**: if a reviewer can't understand the possible routes from the diagram plus one concise
procedure description, the procedure is doing too much.

### Process-procedure best practices

Use only the variables needed for the decision; initialize every follow-up result deliberately; treat
outputs after an unsuccessful action as possibly empty or unvalidated; make null handling explicit;
keep the code deterministic and side-effect-free where possible; give it a business-purpose name; unit
test every decision combination and boundary; inspect the generated process-action code (**Show process
action code**) rather than trusting it compiled; avoid re-querying the same data repeatedly — load one
decision state through an action/subroutine and map it into variables once; keep actual mutation outside
the routing procedure unless there's a narrow, documented reason not to.

## Process-variable design

Use a process variable when a value must cross an action boundary or forms part of the flow's public
contract: passing an action's output into a later input, preserving the current business record ID,
carrying request/response data between connector/parse/persist actions, a status code/page token/file
path/correlation ID/attempt count, a process-procedure decision input, subflow input/output, or a
deep-link/API request/response property. Use a plain constant for a value fixed to one action and not
reused; use ordinary database state when information must survive beyond the process instance, be
queried by others, or support recovery after interruption — a process variable is not durable storage.

### Name by meaning, not by action position

Good: `customer_id`, `request_body`, `http_status_code`, `next_page_token`, `correlation_id`,
`last_successful_sync_at`. Weak: `var1`, `output2`, `temp`, or `response` when several calls exist in
the same flow. Use request/response prefixes or the external system's own name when ambiguity exists.

### Minimize scope

Map only the outputs later actions actually need; mark only genuine subflow inputs/outputs; expose
deep-link/API properties only as part of an intentional public contract; limit process-procedure
availability to real decision inputs and intended outputs; avoid a variable that duplicates durable
database state unless a consistent snapshot is genuinely required. Every additional exposed parameter
enlarges the generated procedure's contract and the Process Flow Monitor's output — least exposure by
default, not "mark everything available" out of habit.

### Initialize and overwrite deliberately

Don't assume a failed action cleared its own output variables. Set response/status variables before
repeated calls or loops. Keep separate `raw_response`, `validated_payload`, and `business_result`
variables when the stages genuinely differ. Use explicit success/valid flags rather than inferring
state from an empty string.

### First-action limitation

For a user-started flow, process variables normally receive values only *after* the first action
completes — they can't feed the first action itself, except on a deep-link start (variables can be
initialized straight from the link/request). Design accordingly:

- Use constants for fixed first-action inputs.
- Let the starting user action produce the outputs later steps need.
- For an autonomous system flow, use a Decision action as the first action, with its process procedure
  loading/initializing whatever state later actions require.
- Use deep-link/API input variables only when that's the documented start contract.

### Large payloads

Variables can carry JSON, XML, files, and response bodies, but for large or long-lived work: store the
payload in a queue/staging table or file store and pass an ID/path instead of copying the same payload
through many variables; consider Monitor/log size and sensitive content; define retention and cleanup
explicitly.

### Secrets in process variables

Access tokens, refresh tokens, and passwords sometimes have to travel between an authentication action
and a call action — treat that path with discipline: load secrets from secure key/configuration
facilities, never hardcode them in action constants or process procedures, keep secret lifetime short
and never expose one as a process/subflow/API output, assume Monitor snapshots and diagnostics may
display mapped variables, don't persist full authorization headers in ordinary logs, redact
request/response bodies containing credentials or personal data, and remember an asynchronous system
subflow runs with `tsf_user()` returning the pool user, not the originating person.

## Subflows

Use a subflow when a coherent action sequence is repeated and has a stable input/output contract —
acquiring/refreshing an OAuth token, sending a standardized notification, generating and archiving a
report, parsing/validating a shared response envelope, claiming/updating a queue item, handling a
common integration error.

Avoid a subflow that exists only to hide two trivial actions, depends on undeclared parent variables,
mixes user and system assumptions, carries many flags that fundamentally change its behavior, or is
directly startable when it was only meant to be called from a parent — restrict starting triggers for
reusable subflows. Mark only required process variables as subflow Input/Output and treat them like a
public function signature.

### Asynchronous system subflows

Use one for long work the parent doesn't need to await — background document generation, fire-and-record
notification, independent enrichment/synchronization. The real consequences: the parent continues
immediately, the async subflow **cannot** return outputs to the parent, it cannot affect parent
continuation, and it runs in system/pool-user context. If the parent must know success, result, or
business identity, don't reach for async output mapping — persist status in durable storage and let the
parent/user observe it separately.

## Transactions and compensation

A process flow orchestrates across action boundaries, UI pauses, connectors, and potentially external
systems — never treat the whole flow as one database transaction. Keep each business mutation atomic
inside its own task/subroutine; commit durable state before relying on it in a later
asynchronous/external action; use idempotency keys for repeatable external calls; use a compensating
action only when reversal is valid and itself auditable; prefer explicit statuses
(Pending/Processing/Completed/Failed) over holding locks across steps; never promise rollback of an
email, file transfer, print, or external API effect — those can't be undone by a database rollback.

## Error handling

Separate these outcomes explicitly rather than collapsing them into one pass/fail: the action executed
technically; the transport succeeded; the remote system returned a successful protocol status; the
payload was syntactically valid; the business outcome was accepted; local persistence completed.

Recommended failure categories:

| Category | Examples | Handling |
|---|---|---|
| Retryable | Timeout, temporary network failure, HTTP 429/selected 5xx, transient lock | Retry with backoff |
| Business rejection | Invalid state/data | Don't retry unchanged input |
| Configuration/authentication | Bad credentials, expired config | Stop/escalate, maybe refresh credentials once |
| Poison payload | Malformed/unrecoverable data | Quarantine with safe diagnostics |
| Internal defect | Unexpected code fault | Fail visibly, log correlation/stack securely |

Every system flow should be able to answer: what is retried, how many times and with what backoff,
what prevents duplicate side effects, where is failed work visible, who is notified, and how is it
resumed or replayed. Don't implement a long sleep inside a flow — store `next_attempt_at` in a queue and
let a later scheduled run claim eligible work instead.

## When to schedule a process flow

Schedule a **system flow** when the work is autonomous and naturally time- or interval-driven: polling
an external system that can't push events, processing a queue in bounded batches, periodic
reference/master-data sync, recurring reports/notifications, cleanup of expired/stale records or files,
cache/materialized-data recalculation during quiet periods, database maintenance/security scans/
retention jobs, or checking deadlines/escalations where crossing time itself creates work.

### When not to schedule

An event, API call, webhook, message broker, or table action can start the work promptly and reliably;
the flow needs user choice or UI state; one plain database scheduled job is simpler and equally
observable; polling would repeatedly scan large tables with no work to do; the timing is really a
business deadline needing durable state/escalation/audit (model the due work explicitly, then schedule
a worker over it); a previous run may overlap and the flow isn't concurrency-safe; or failure would be
invisible with no owner monitoring it.

**Scheduling is a trigger, not a reliability strategy.** The flow still has to be idempotent,
restartable, observable, and bounded regardless of how it's triggered.

### Choosing frequency

Choose it from the business service level and workload, not the shortest available interval:

| Need | Typical reasoning |
|---|---|
| Near-real-time queue | Short interval, cheap empty check, bounded batch |
| External sync | Source change rate, API limits, acceptable staleness |
| Daily report | After source data closes, before recipients need it |
| Cleanup | Retention requirement and storage growth |
| Maintenance | Low-activity window, after large nightly writes where appropriate |
| Deadline/escalation | Fine enough not to miss the service-level boundary |

Also weigh: expected and worst-case run duration; whether a new run could start before the previous
finishes; database/external API load and rate limits; time zone/DST and IAM's UTC schedule logs;
maintenance/deployment windows; dependencies between jobs; backlog recovery after downtime; and any
per-customer/per-environment custom-schedule need.

### Multiple running instances

Keep disabled unless parallelism is both necessary and safe. Enable only when work is partitioned or
claimed atomically, each item has an idempotency key, external/database capacity supports concurrency,
ordering isn't required (or is enforced per partition), and a lease/heartbeat/recovery strategy exists.
Don't enable it merely because one run is slow — reduce batch size, optimize queries/calls, or use a
queue first. The platform doesn't let you cap the exact number of parallel instances — use durable
queue/claim logic to control effective concurrency instead.

### Custom schedules in IAM

Allow IAM custom schedules when runtime administrators legitimately need environment/customer-specific
timing (local business hours/time zones, different API-rate contracts, different data volumes,
customer-specific delivery, staggering load across tenants). Keep the model schedule authoritative
instead when timing is part of the product's correctness or administrators could create unsafe overlap.
Document purpose, expected duration, dependencies, and safe frequency in the system flow's own
description, kept in sync with what IAM shows.

## Starting triggers as a public contract

Process flows can start via user actions, deep links, custom protocols, or APIs — enable only the
triggers actually intended, especially for subflows. For API/message-protocol flows specifically: map
request method/path/query/headers/body only into the typed variables actually needed; return an
explicit response code, headers, and body; authenticate/authorize appropriately, with `/open`-style
endpoints getting extra scrutiny; validate size, content type, schema, and replay/idempotency; never
trust request variables in SQL composition (see `SKILL.md`'s connector guidance); and keep versioning/
backward compatibility in mind — every variable/property mapping here is effectively part of an API
contract once published.

## Testing strategy

- **Model tests**: every action has intentional input/output mapping; every action has defined
  success/failure continuation; no unreachable actions or accidental direct subflow triggers exist;
  variables use correct domains and minimal exposure flags; rights and starting triggers match the
  intended audience.
- **Process-procedure tests**: every variable combination enables the correct route/order; null/
  unvalidated outputs after failure are handled; exactly the intended paths remain enabled; unknown
  status codes fail safely; the procedure has no accidental side effects.
- **Integration tests**: success, timeout, 4xx, 429, 5xx, malformed payload, and business rejection;
  token expiry/refresh and invalid credentials; duplicate/replayed request; partial page/batch failure;
  rate limiting and large payload; external success followed by local persistence failure.
- **Schedule and concurrency tests**: manual "Run now"; empty run and large backlog; runtime longer
  than the schedule interval; concurrent starts with multiple instances both off and on; atomic queue
  claim and stale-lease recovery; downtime/restart/backlog behavior; time-zone/DST boundaries; custom
  IAM schedule override/reset.
- **User-flow tests**: cancel, close, back navigation, and validation failure; Add/Edit start behavior
  with Auto-save disabled; insufficient permissions for a later action; refresh/open-document behavior;
  Process Flow Monitor state at every action.

## Common anti-patterns

- **Flow for one task** — needless orchestration around one cohesive operation.
- **Rigid wizard for exploratory work** — users can't deviate from a sequence that isn't actually fixed.
- **Business logic hidden in process procedures** — the diagram lies about what actions do.
- **Every variable exposed everywhere** — generated contracts and monitor snapshots become noisy/risky.
- **Variable as durable state** — a restart loses the only record of progress.
- **Stale output after failure** — routing uses a value left over from an earlier iteration/action.
- **First-action variable assumption** — a user-started flow expects an input that's actually
  uninitialized on entry.
- **SQL interpolation** — connector SQL built from raw variable text instead of parameters.
- **Error message equals failure** — routing assumes a task failed because it showed an error message.
- **One global transaction assumption** — external effects that can't actually be rolled back.
- **Retry without idempotency** — duplicate orders, files, emails, or postings.
- **Tight scheduled polling** — frequent empty runs burning resources for no real SLA benefit.
- **Multiple instances as a performance fix** — workers racing on the same data.
- **No-work treated as an error** — normal queue emptiness flooding logs/alerts.
- **Poison item blocks the queue** — one invalid payload retried forever.
- **Async subflow with an expected output** — the parent continues without the result it actually needs.
- **Generic action names** — dozens of `task`/`decision`/`web_connection` boxes obscuring intent.
- **Failure ends silently** — administrators can't see or replay rejected work.

## Recommended implementation workflow

1. Define the trigger, user/system context, outcome, and service level.
2. Decide whether orchestration is actually needed, or one task/subroutine is enough (see "When not to
   use a process flow" above).
3. Draw the main success path and explicit terminal states.
4. Add action-level failure paths before optional branches.
5. Choose durable database state for recovery, and process variables only for transient handoff.
6. Type and name the minimum variable set.
7. Use modeled step conditions for simple routing; add a process procedure only for
   variable-dependent immediate routing/order.
8. Extract repeated coherent sequences into typed subflows.
9. Add idempotency, retry classification, queue leases, and compensation where needed.
10. If autonomous, choose event/API triggering first; schedule only when time/polling genuinely fits.
11. Define schedule frequency, overlap policy, batch size, load window, and custom-IAM policy.
12. Test every success/failure branch, concurrency case, restart, and large-data scenario.
13. Document operational ownership, replay procedure, and safe manual execution.

## Final review checklist

- Does the work have a meaningful ordered sequence, or would one task/subroutine do?
- Is the flow correctly classified as user or system flow?
- Is the main path concise and visually understandable?
- Are Success, Not-successful, and Always connections deliberate — not defaulted?
- Does every failure end in retry, rejection/quarantine, compensation, or a visible failure state?
- Are process procedures limited to immediate routing/order logic, not core mutation?
- Can the simple routing be understood from the diagram alone?
- Are variables used only for transient handoff/public contracts, typed, named by meaning, initialized,
  and minimally exposed?
- Is the first action valid for the chosen start trigger (constants/deep-link/Decision-init)?
- Are secrets and large payloads handled safely?
- Are subflow inputs/outputs explicit, and is async execution used only when the parent needs no result?
- Are database actions atomic and external effects idempotent?
- If scheduled: is the flow fully autonomous, is scheduling actually preferable to an event/API
  trigger, are frequency/time-zone/overlap/downtime/workload accounted for, and is multiple-instance
  execution proven concurrency-safe rather than assumed?
