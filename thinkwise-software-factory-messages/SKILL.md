---
name: thinkwise-software-factory-messages
description: Reference guide for creating and maintaining modeled user-facing messages in a Thinkwise Software Factory model — errors, warnings, confirmations, process-flow choices, progress text, and translated database errors. Use whenever an MCP connector with Software Factory access creates, inspects, or troubleshoots a message, or before writing a message-calling SQL statement or a database-error-capture regex.
---

# Creating and Maintaining Messages in the Thinkwise Software Factory

Reference for the full message lifecycle: `msg` (master object) → `msg_option` (choice messages only) →
translation (one `transl_object_transl` row per application language) → wired into SQL
(`tsf_send_message`/`tsf_send_progress`), a task's confirmation setting, a process flow's **Show
message** action, or a database-error-capture rule. Domain key verified live: **`sf/manage_messages`**
holds `msg`, `msg_option`, `audio_file`, plus read access to `transl_object_transl`/`process_action`/
`task`/`icon` for usage lookups.

## What a modeled message is, and what it isn't

A modeled message is a reusable contract between business logic and the UI: logic emits a stable
`msg_id` plus optional parameters; the runtime resolves translation, presentation, severity, and
behavior. It is **not** the same thing as:

- Application-domain records such as inbox messages, chat messages, or notifications — those are
  ordinary tables in your data model, not `msg`.
- Software Factory validation findings (`validation_msg`) — the modeler's own lint output, unrelated.
- Message-broker payloads (`message_broker`/`message_broker_message`, MQTT etc.) — integration data on
  the wire, not UI text. See `thinkwise_software_factory_process_flows`'s message-broker action types
  for that.
- Server logs and technical diagnostics not modeled under **User interface > Messages** — use
  structured server-side logging for those instead.

```text
Business logic / process flow / database error
                    |
                    v
          msg_id + parameter XML
                    |
                    v
     msg row + active-language translation
                    |
                    v
       Popup, panel/snackbar, debug, or suppress
```

## Entity map (`sf/manage_messages`)

| Entity | Key (adds to parent) | Purpose |
|---|---|---|
| `msg` | `msg_id` | Master object: location, severity, database-capture config, audio |
| `msg_option` | `msg_option_id` | One row per choice on a **Show message** process action |
| `audio_file` *(read-only)* | `audio_file_id` | Uploaded `.wav`/`.mp3` assets — uploaded elsewhere (Software Factory file/theme management), only referenced here |

`msg` and `msg_option` are both directly writable (`allow_add/update/delete = true`) — no
creation-order gate, same shape as `subroutine` in `thinkwise_software_factory_subroutines`.

### `msg` fields (verified)

`msg_description` (developer-facing, not shown to users), `msg_location_id` (**string** enum — the
column literally holds `"popup"`/`"panel"`/`"suppress"`/`"debug"`, not a numeric code), `severity`
(**byte** enum: `error` 0, `warning` 1, `information` 2 — don't confuse this with the *database*
severity levels 9/16 mentioned under "Abort behavior" below, which are a SQL Server engine concept,
unrelated to this field), `msg_error_code`, `msg_regular_expression`, `priority` (byte — lower number
wins when multiple capture rules match), `audio_file_id`, `generated_by_control_proc_id`.

### `msg_option` fields (verified)

`response_type` (**bool flag** — `true` = affirmative, `false` = negative, confirmed live),
`msg_option_status_code` (int32 — see the status-code convention below), `icon_id`/`icon`, `order_no`.
Set `icon_id` to a suitable icon per `thinkwise_software_factory_icons` as part of creating each
option — reinforce the outcome (Continue → check/arrow-forward, Retry → retry arrow, Cancel → X) rather
than leaving it unset, and keep affirmative/negative icons visually distinct from each other.

**Status-code convention, verified against the base model's own confirmation messages**: affirmative
options (`response_type = true`) get status codes `0`, `1`, `2`, … in ascending order; negative options
(`response_type = false`) get `-1`, `-2`, `-3`, … Real example (`confirm_set_branch_close_all_documents`):
`no` → `response_type=false, status_code=-1`; `yes` → `response_type=true, status_code=0`; `yes_always`
→ `response_type=true, status_code=1`. Follow this convention on every new choice message — a process
flow branching on the numeric status code (see "Wiring into a process flow" below) depends on it being
predictable.

## Choosing severity and location

| Situation | Severity | Location | Abort? |
|---|---|---|---|
| The requested operation is invalid | Error | Popup | Yes |
| The operation is valid but risky | Warning | Popup | Usually no, or ask confirmation first |
| The action completed successfully | Information | Panel/snackbar | No |
| The user must choose how a process continues | Information or warning | Popup with `msg_option`s | Controlled by the selected status code |
| Long-running task progress | Information | Progress display (`tsf_send_progress`) | No |
| Raw database constraint error | Usually error | Popup after capture | Yes |
| Expected noisy database message | Any | Suppress | Depends on the original action |
| Developer-only diagnostic | Information | Debug/log | No |

**Error** — use when the change/operation cannot safely complete; Thinkwise documents that only error
severity cancels the action at the platform level. Say what could not be done, why in business
language, and what the user can change next. Never expose SQL, table/constraint names, stack traces,
credentials, endpoints, or personal data — keep those in secured logs with a correlation id.

**Warning** — the action is allowed but has an unusual/harmful/irreversible consequence, and the warning
must support a real decision. If the condition actually makes execution invalid, that's an error, not a
warning. If there's no meaningful choice, use information instead of forcing an acknowledgement.

**Information** — successful completion, useful status, or neutral explanation. Prefer panel/snackbar
for short-lived feedback; reserve popup for information that must be read before continuing. Avoid a
success message after every ordinary save — the updated screen state is usually confirmation enough.

## Message location

- **Popup** — blocking errors, important warnings, confirmations, process-flow choices. Reserve for
  what genuinely needs attention; popups interrupt work.
- **Panel/snackbar** — bottom-of-screen in the legacy Windows GUI, a snackbar in Universal UI.
  Non-blocking feedback, status updates, sequences of related messages. Two reserved messages manage
  panel output from code, verified live in the base model (`add_separator`, `msg_location_id='panel'`,
  "insert a separator between previous and new panel messages"; `clear_panel`, same location, "clear
  the previous messages") — Thinkwise explicitly advises using these only from code, never as a task
  confirmation message or a Show message process action.
- **Suppress** — a known database message that should not reach the user. Changes presentation only,
  not correctness — document *why* it's safe, since suppression can turn a visible defect into an
  unexplained silent failure.
- **Debug** — primarily a legacy-client facility; prefer structured server-side logging with
  correlation ids for current production apps. Never depend on a debug message for essential guidance.

**Audio** — `.wav`/`.mp3` on popup/panel messages via `msg.audio_file_id` (references an existing,
separately-uploaded `audio_file` row — this domain can't create one). Use only where visual attention
is insufficient (hands-busy shop floor, a safety alert); always pair with a visual equivalent and
respect shared workspaces/accessibility.

## Plan first

Applies `thinkwise_software_factory_mcp_base`'s "Confirm-before-mutate" convention (see its Shared
conventions section) to a message — don't restate that rule, apply it. Before the first
`stage_resource`/`stage_task` call that creates a `msg`, `msg_option`, or translation row, present the
user with:

- The proposed **severity**, **location**, and **abort behavior** — the decision step 1's table below
  leads to, stated as a concrete recommendation rather than left implicit.
- The **drafted message text**, including its parameters in order (see "Writing effective message
  text" and "Parameters and translated text" below for how to draft it) — and, if this is a choice
  message, the drafted `msg_option` labels and status codes too.

Get explicit confirmation on that package before creating anything. If the answer changes the
severity/location/abort choice or the text itself, re-confirm the updated version rather than staging
the original draft.

## Step-by-step: creating a message

1. Decide severity, location, and abort behavior first — everything else follows from that (use the
   table above). This is the decision "Plan first" above presents to the user before step 2 creates
   anything.
2. **Create the `msg` row**: purpose-based `msg_id` (see "Naming and reuse" below), `msg_description`
   for developers, `msg_location_id`, `severity`. Leave `msg_error_code`/`msg_regular_expression`/
   `priority` unset unless this is a database-capture message (see below).
3. **Write the source-language translation.** A `msg` is a translation object (`type_of_object = 4`,
   verified live) — its translated text lives in `transl_object_transl`, one row per application
   language, matched on `(type_of_object=4, transl_object_id=<msg_id>, appl_lang_id)`. Follow
   `thinkwise_software_factory_translation_objects`'s generic workflow for finding/overwriting the
   `[bracketed]`-placeholder row rather than adding a new one — this skill only adds the `msg`-specific
   fact that its `type_of_object` is `4` and its live text field is `transl`.
4. **If this is a choice message (Show message with options)**, add `msg_option` rows — see "Message
   options" below, including the shared-translation gotcha before you name a new option.
5. **Add every other application-language translation** and get each reviewed.
6. **Wire the caller** — SQL (`tsf_send_message`), a task's confirmation setting, a process flow's Show
   message action, or a database-capture rule. See the sections below for each.
7. **Copy/rename/delete** via the bound tasks on `msg`, verified: `task_copy_msg` (`from_msg_id`,
   `to_msg_id`), `task_rename_msg` (`branch_id`, `from_msg_id`, `to_msg_id`), `task_delete_msg`
   (`branch_id`, `msg_id`). Prefer these over hand-duplicating a message.
8. **Before renaming/deleting/changing placeholders on an existing message**, check its usage —
   `msg`'s navigation properties `detail_ref_msg_process_action` (process actions using it),
   `detail_ref_msg_task` (tasks using it as a confirmation message), and
   `detail_ref_msg_task_variant_overview` (task variants) all resolve live; query them before touching
   an established message's contract.

## Calling a message from SQL

Verified live (SQL Server, real parameter names from a live model — not the generic placeholder names
docs sometimes show):

```sql
exec dbo.tsf_send_message
    @msg_id        = 'customer_blocked',
    @parmtr_string = @parameter_xml,
    @abort_ind     = 1;
```

`tsf_send_message`'s real parameters, in order: `msg_id`, `parmtr_string` (the parameter XML, see
below), `abort_ind`. This is itself an ordinary subroutine (see
`thinkwise_software_factory_subroutines`) — call it exactly like any other, with named parameters.

### Abort behavior

On SQL Server: `abort_ind = 0` continues the flow (database severity level 9); `1` or `NULL` treats the
action as reversed (database severity level 16) — these severity *levels* are a SQL Server engine
concept triggered by the call, distinct from the modeled `msg.severity` enum above. **Calling an
aborting message does not automatically roll back a SQL Server transaction.** Logic that must stop
should roll back and return explicitly:

```sql
exec dbo.tsf_send_message @msg_id = 'customer_blocked', @parmtr_string = null, @abort_ind = 1;
rollback;
return;
```

In a `try/catch`, an aborting message transfers control to the catch block — design the transaction and
error-handling path together, or code may continue, partially commit, or replace the useful modeled
message with a generic exception.

**Database-platform behavior differs — don't assume SQL Server behavior is portable.** Oracle raises an
application error for aborting messages; DB2 always aborts a sent message and additionally provides
`v_message_text` for informational output specifically inside Defaults/Layouts. Check
`branch_rdbms_type` (see `thinkwise_software_factory_create_control_procedures`) before writing an
abort/rollback pattern and confirm against the target platform's real behavior, not just the SQL Server
default above.

### Progress messages

For long-running SQL tasks, `tsf_send_progress`'s real parameters, verified: `msg_id`, `parmtr_string`,
`percentage`.

```sql
exec dbo.tsf_send_progress
    @msg_id        = 'orders_processed',
    @parmtr_string = @parameter_xml,
    @percentage    = 40;
```

`-1` displays indeterminate/marquee progress; `0`–`100` displays determinate progress. Use determinate
progress only when the denominator is reliable — don't run an expensive recount solely to feed the bar.
Update at meaningful intervals, not every row, and never present `100%` before the transaction/
downstream work has actually finished.

A third framework subroutine, `tsf_send_assertion_msg` (verified live, parameters `success_ind`,
`parmtr_string`), exists alongside these two — an assertion-style helper distinct from either.

## Parameters and translated text

Translations use positional placeholders `{0}`, `{1}`, … The caller supplies XML elements in that
order. **Always build the XML safely** — `for xml path('text')` (or an equivalent safe builder) escapes
`&`, `<`, `>`, quotes, and apostrophes; hand-built string concatenation breaks on ordinary business
values like `R&D` and can create injection or malformed-message problems:

```sql
declare @parameters nvarchar(500) = concat(
    (select @customer_name for xml path('text')),
    (select @order_number  for xml path('text'))
);

exec dbo.tsf_send_message @msg_id = 'order_cannot_be_released', @parmtr_string = @parameters, @abort_ind = 1;
```

Supported parameter elements: `<text>` (literal runtime value), `<tab>`/`<col>` (translated table/
column label), `<domelement>` (translated domain-element label), `<task>`/`<taskparam>`,
`<report>`/`<reportparam>`. Use a translated-object reference when the message should show the same
localized label the UI already uses for that object; put literal runtime values in `<text>`.

Best practices:
- Keep placeholder order consistent across every language; a translator may still reorder `{0}`/`{1}`
  in their own sentence — check the reordering still reads correctly.
- Put the complete sentence in the translation; don't concatenate translated fragments in SQL.
- Include only values that help the user identify or fix the problem.
- Format dates/numbers/quantities/currencies for the user's locale when the runtime doesn't already.
- Test null, empty, long, Unicode, and XML-special-character values.
- Never embed secrets or sensitive identifiers.
- Prefer one meaningful aggregate message over one popup per row in bulk processing — see
  "Task confirmation messages" below for the modeled control that prevents exactly this.

## Message options — and a shared-translation gotcha

Each `msg_option` has a translated option name, a `response_type` (affirmative/negative), a
`msg_option_status_code`, an optional icon, and `order_no`. Use options when the decision is small and
discrete; use a task popup when the user must enter structured data; use a separate screen when the
decision needs substantial context.

**Verified live, and easy to get wrong: an option's translated label is keyed by its bare
`msg_option_id` alone — not scoped by `msg_id` — so every message that uses `msg_option_id = 'yes'`
shares the exact same `transl_object_transl` row (`type_of_object = 496`, `transl_object_id = 'yes'`).**
Confirmed by reading the identical row (same `transl` text "Yes") under two entirely unrelated messages'
`yes` options. This means:
- Reusing `yes`/`no` across many messages is fine, even desirable, when the generic "Yes"/"No" label is
  correct — it's one translation to maintain, and it stays consistent app-wide.
- **Editing the shared `yes`/`no` translation to fit one specific message's wording changes it on every
  other message using plain `yes`/`no` too**, silently. If a message needs its own distinct wording
  (e.g. "Release anyway" instead of a generic "Yes"), give that option a distinct `msg_option_id`
  entirely (verified real examples from the base model: `yes_continue`, `yes_always`) — don't reuse
  `yes` and then edit its shared translation.

Best practices for option text:
- Label with verbs and outcomes: **Release anyway**, **Return to planning**, **Cancel** — not bare
  "Yes"/"No" once the consequence needs to be explicit.
- Give affirmative/negative options a real correspondence with process-flow green/red arrows (see
  below); don't make the user guess from button position alone.
- Handle close, cancel, timeout, and unexpected status values downstream.
- Keep the default/focused option safe for destructive actions.
- Test every route, including messages with multiple affirmative *and* multiple negative options.

## Wiring into a process flow — the Show message action

Verified live against `process_action`'s metadata (this isn't yet documented in
`thinkwise_software_factory_process_flows` — cross-reference this section from there):

- **Action type**: `process_action.process_action_type = show_msg` (enum value `350`) — the docs/
  research sometimes call this "Show message"; the modeled enum id is `show_msg`.
- **Which message**: set `process_action.msg_id` directly on the action.
- **Routing on the chosen option, precisely**: `process_step.last_process_action_successful` (the
  ordinary `not_successful`/`successful`/`always` step condition used by every action type) is too
  coarse for a message with more than one affirmative or more than one negative option — it can't
  distinguish `yes` (`0`) from `yes_always` (`1`), for instance. To route on the **exact numeric status
  code**, capture it into a process variable and branch on that instead: the `show_msg` action has a
  pre-seeded `process_action_modeler_fixed_output` row keyed `output_parmtr_id = 'status_code'`
  (verified — this is a generic output present on the fixed-output enum, not `msg`-specific naming).
  Following the same pre-seeded/edit-only pattern documented in
  `thinkwise_software_factory_process_flows`'s "Wiring runtime values" section: query for that existing
  row (filtered by `process_action_id`), edit it to set `process_variable_id` to a matching-domain
  `process_variable`, then follow the action with one or more `decision` actions testing that variable's
  value against the specific `msg_option_status_code`s to route each branch.
- For a genuinely binary yes/no message, the plain `last_process_action_successful =
  successful`/`not_successful` step condition is enough — affirmative maps to `successful`, negative to
  `not_successful` — and the status-code capture above is unnecessary.

Keep the technical work in tasks/subroutines and let the process flow coordinate the user interaction —
a `show_msg` action should only ever present and branch, never itself contain business logic (see
`thinkwise_software_factory_process_flows`'s "Process logic and control procedures" section for where
that logic actually belongs).

## UX principles for confirmations and choices

Two forces have to hold at once, and they pull against each other: **interrupt rarely** (a dialog
that fires on routine actions trains people to dismiss it without reading — and then they dismiss the
*next* one too, the one that mattered) and **when you do interrupt, be specific** ("Are you sure?"
gives the user nothing to check their intent against). Applying this to `msg`/`msg_option` design:

- **Label options by outcome, not Yes/No** — "Release anyway" / "Back to edit", not a bare Yes/No a
  user has to reconstruct meaning from. Already the convention for `msg_option` text (see "Best
  practices for option text" above); stated here as the general principle behind it.
- **Don't default to the dangerous answer.** Prefer no default at all; if the platform forces one,
  make the safe option the default and don't let the destructive option sit as the reflexive click.
- **For rare, genuinely catastrophic actions, require a deliberate act rather than a click.** A
  confirmation dialog becomes muscle memory the moment it's routine. For the few truly irreversible
  actions (wiping an environment, dropping a list), make the user do something they wouldn't do by
  reflex — e.g. type the object's name into a task parameter the task validates before proceeding.
  Reserve this for genuinely severe cases; using it everywhere just creates a new reflex.
- **Let an educational confirmation be switched off.** If a confirmation exists mainly to teach a
  feature's side effect the first few times, it should be dismissible so it doesn't graduate into
  permanent noise. Thinkwise has no built-in per-user "don't ask again," so treat this as a product
  decision to raise with the user, not a message-model feature to assume exists.

### When you can't infer severity/consequence, ask

Purpose and severity are almost always inferable from the request and the model — "add a confirmation
to the Delete Customer task" tells you the purpose, and the data model tells you the consequence (does
it cascade, how many dependent rows). Ask the user (the "Ask, don't default" convention from
`thinkwise_software_factory_mcp_base`, applied here) only when:

- **Whether the action is serious enough to interrupt at all isn't visible in the model** — is this
  "delete" a hard delete or a soft archive; is this "send" reversible?
- **The consequence itself isn't visible** — cascade depth, dependent-row count, what the user stands
  to lose. Guessing here produces exactly the vague dialog this section warns against.
- **The message branches and the branches' actual behavior isn't specified yet** — an
  affirmative/negative choice is meaningless until each outcome is known.

Ask the narrow factual question (the consequence, the severity, what each branch does), not "what
should the message say" — that's the writing, which follows once the facts are in hand.

## Task confirmation messages

A task can request confirmation before execution — verified fields on `task`: `ask_confirmation` (bool),
`confirmation_msg_id` (FK to `msg`), and `popup_for_each_row` (bool). Use confirmation for destructive,
expensive, broad, or externally visible actions — not as a substitute for a clearly named button.

**Include at least one parameter for context whenever practical.** A confirmation message with zero
parameters is flagged by a Software Factory validation — a generic "Are you sure?" that names nothing
specific about the record or action is exactly the vague text "Writing effective message text" above
warns against.

**`popup_for_each_row` is the modeled control for the bulk-confirmation anti-pattern** ("message storms"
in the failure patterns below) — when a task can run against multiple selected rows, decide deliberately
whether confirmation should fire once for the whole batch (`popup_for_each_row = false`, the usual
choice) or per row (`true`, rarely what you want — it's the mechanism behind the very message-storm
problem to avoid, so flip it on only when a genuinely per-row decision is required).

A dynamic confirmation (values filled in at runtime) is built by:
1. Adding task parameters for the values to display.
2. Filling those parameters in the task's Default control procedure.
3. Referencing them (via `<taskparam>`, see above) in the confirmation message's translation.

Good confirmation text identifies the action *and* consequence:

> Release order {0}? This will create {1} production jobs.

Weak confirmation text merely repeats the button ("Are you sure?"). Don't use confirmation when the
system can already determine the action is invalid — disable it or show an error instead. Ensure
keyboard focus and button labels make the safe choice clear.

## Database message capture

A modeled message can recognize a raw database engine error (unique index, foreign key, check
constraint, …) and replace it with localized business text, via `msg_error_code` +
`msg_regular_expression` + `priority` + named capture groups the translation can reference (e.g.
`{constraint}`).

Real verified examples from the base model (SQL Server), showing the actual named-group syntax and that
the framework's own generic captures sit at `priority = 100`:

```text
msg_id: mssql_error_duplicate_key_row       error_code: 2601   priority: 100
regex:  Cannot insert duplicate key row in object '(?<table>.+)' with unique index '(?<index>.+)'\.
        The duplicate key value is \((?<value>.+)\)\.

msg_id: mssql_error_check_constraint_insert  error_code: 547    priority: 100
regex:  The INSERT statement conflicted with the CHECK constraint "(?<constraint>.+)"\. The conflict
        occurred in database "(?<database>.+)", table "(?<table>.+)"(, column '(?<column>.+)')?\.
```

**A narrower, business-specific capture on the same error code needs a lower `priority` number than
100 to actually win** — e.g. `priority = 10` for a message translating the exact unique-index violation
on `customer.email` into "A customer with this email already exists," layered on top of (not replacing)
the generic `mssql_error_duplicate_key_row` catch-all. See `references/practical_examples.md` for a
worked version of this pattern.

### Recommended design

1. Keep the database constraint as the final integrity guarantee — don't remove it just because a nicer
   message now exists.
2. Where practical, validate earlier and present a contextual business error before the write is even
   attempted.
3. Add a capture message as the fallback for races, alternative write paths, and direct constraint
   failures the earlier validation didn't catch.
4. Match the narrowest stable signature available — prefer an exact error code plus a narrow expression
   over a broad catch-all.
5. Translate technical identifiers (constraint/index/table names) into user-recognizable business
   concepts; don't expose them raw just because they're available in a capture group.

### Regex guidance

- Anchor stable portions of the message where possible.
- Use named groups only for values that actually improve the user-facing message.
- Assign more specific patterns a lower priority number so they win over a generic base-model capture.
- Test against the actual database engine and version in use — error text differs across SQL Server
  versions and RDBMS platforms.
- Include near-miss error samples to prove the expression doesn't capture unrelated faults.
- Recheck patterns after a database-platform upgrade or a localization change to the engine's own error
  text.

## Messages by logic concept

- **Defaults and layouts** — these fire frequently, sometimes mid-typing. Avoid popups from routine
  default/layout evaluation; use validation state, field visibility, mandatory state, or task-button
  state instead. If an informational message is essential here, make sure it can't repeat on every
  refresh.
- **Tasks and subroutines** — a natural place for completion, validation, and integration messages. A
  reusable subroutine (`thinkwise_software_factory_subroutines`) should normally return a structured
  result to its caller and emit a user message only when its contract explicitly owns presentation — a
  subroutine that always pops a message becomes hard to reuse from an API, a job, or a process flow.
- **Triggers and handlers** — use messages here for database-enforced failures that must stay consistent
  across every write path. Keep wording business-oriented, roll back deliberately, and support
  multi-row operations — never one message per affected row.
- **Process logic** — prefer the standard modeled message mechanism and predictable abort behavior for
  errors returned from CRUD processing; make sure the message actually corresponds to the transaction
  outcome.
- **Process flows** — use **Show message** for presentation and branching (see above); keep the real
  technical work in tasks/subroutines and let the flow coordinate the interaction.

## API, offline, and unattended behavior

A Thinkwise application isn't always driven through a GUI — the same logic can run through Indicium
APIs, scheduled work, process flows, or offline sync.

- Never make correctness depend on a user clicking a popup.
- Return a stable error condition and a suitable HTTP/process outcome for aborting errors.
- Assume information/warning presentation differs by client.
- Never require an interactive confirmation on an unattended execution path.
- Separate a machine-readable result from the localized user-facing text where an integration needs to
  react programmatically — don't make a caller parse translated prose to determine the error type.
- Test the actual Indicium response, not only the Universal UI rendering.

## Naming and reuse

Lower-case, descriptive `msg_id`s naming the condition or result: `order_cannot_be_released`,
`customer_already_exists`, `planning_completed`, `integration_temporarily_unavailable`. Avoid `msg_001`/
`error_2` (no business meaning), `task_order_message` (describes location, not condition),
`something_went_wrong` (not actionable), or reusing `record_not_found` for unrelated objects that need
different remedies.

Reuse a message only when semantics, severity, location, parameter contract, *and* remedy are genuinely
identical — create a separate message the moment the user needs different guidance, even behind the
same underlying technical exception. Treat the `msg_id` and its parameter order as an interface: check
`detail_ref_msg_process_action`/`detail_ref_msg_task`/`detail_ref_msg_task_variant_overview` (its actual
usage) before renaming, deleting, or reordering placeholders.

- **If it's ambiguous whether an existing message can be reused, or whether a new `msg_option` should
  share `yes`/`no` (see the shared-translation gotcha above) versus get its own distinct id, ask the
  user rather than guessing compatibility** — per `thinkwise_software_factory_mcp_base`'s "Ask, don't
  default" convention (see its Shared conventions section).

## Writing effective message text

Pattern: **Action/subject + problem or result + next step.**

- "Order {0} cannot be released because no routing is configured. Add a routing and try again."
- "Import completed: {0} rows added and {1} rows skipped. Open the import log for details."
- "The planning run will replace {0} manually scheduled operations. Continue?"

Guidelines: use the user's vocabulary, not database vocabulary; state the affected object when
ambiguity is possible; make remediation specific; avoid blame ("You entered…"); avoid unexplained
abbreviations; no period after a one-word button label; keep primary text short and point to a log/
subject for large detail sets; for bulk work, summarize counts and provide a reviewable exception list
rather than one message per row.

## Security and privacy

Never expose SQL text, connection strings, tokens, credentials, stack traces, or internal paths.
Minimize personal/commercially sensitive data in messages and logs. Avoid confirming whether an
unauthorized record exists. Sanitize values originating from external systems. Keep detailed
diagnostics in access-controlled logs, using correlation ids that support investigation without
revealing internals. Remember messages may be captured in browser logs, screenshots, monitoring, or API
responses — don't put anything in one you wouldn't put in a log line visible to a wider audience.

## Unit testing

Load `thinkwise_software_factory_unit_tests` for the full mechanics — it already covers the
`unit_test_msg` entity ("expected message(s) for the sad-flow/validation path") as part of any code
type's unit test. At minimum, cover: the message appears for the exact invalid condition; boundary/valid
cases do *not* emit it; abort vs. non-abort behavior matches the design; transactions fully roll back on
failure; a bulk operation produces one useful summary rather than a message storm; parameters handle
null/Unicode/max-length/XML-special-character values. For a database-capture message specifically, also
verify: the regex matches representative raw errors, near-misses don't match, priority selects the most
specific message when several patterns could apply, and named groups populate the correct placeholders.

## Common failure patterns

See `references/practical_examples.md` for worked examples alongside this list: popup overuse (routine
success/layout refreshes interrupting the user), message storms (row-by-row validation in a bulk
action — check `popup_for_each_row` first), wrong severity (a warning used where continuing would
violate an invariant), abort without rollback, rollback without return (later code runs and hides the
original failure), hand-built XML breaking on `&`/`<` in real data, leaked implementation details
(index/constraint ids shown directly), a broad capture regex catching unrelated errors, unstable
priority (a generic base-model capture winning over a specific business rule), translation drift
(languages with different/missing placeholders), vague confirmation text, presentation logic baked into
a reusable subroutine, success noise on every save, suppression used as a fix for a real defect, and an
integration parsing localized prose instead of a stable status code.

## Pre-flight checklist

- Create through `sf/manage_messages` (`msg`/`msg_option`/read-only `audio_file`); no creation-order
  gate to worry about.
- `msg.msg_location_id` holds the literal strings `popup`/`panel`/`suppress`/`debug` — don't look for a
  numeric code there; `severity` *is* a numeric byte enum (`error`=0/`warning`=1/`information`=2).
- Set `msg_option.icon_id` to a suitable icon per `thinkwise_software_factory_icons` for every option —
  don't leave it unset by default.
- Don't confuse `msg.severity` with the SQL Server *database* severity levels (9/16) that
  `tsf_send_message`'s abort flag triggers — same word, unrelated concepts.
- Call `tsf_send_message`/`tsf_send_progress` with their real parameter names
  (`msg_id`/`parmtr_string`/`abort_ind` and `msg_id`/`parmtr_string`/`percentage`), named, like any other
  subroutine call.
- Build parameter XML with `for xml path('text')` or an equivalent safe builder — never hand-built
  string concatenation.
- Aborting a message does not roll back a SQL Server transaction by itself — write the explicit
  `rollback; return;` (or platform equivalent) yourself, and confirm the actual behavior on non-SQL
  Server platforms rather than assuming it matches.
- Message options: `response_type` true=affirmative/false=negative; status codes ascend from `0`
  (affirmative) and descend from `-1` (negative) — follow this even for a brand-new choice message.
- **Before naming a new `msg_option_id`, check whether reusing `yes`/`no` is actually fine** — its
  translation is shared by every message using that same bare option id. Give a message-specific label
  its own distinct option id instead of editing the shared `yes`/`no` text.
- A process flow's Show message action is `process_action_type = show_msg` (350); route on the exact
  chosen option via the pre-seeded `process_action_modeler_fixed_output` row for
  `output_parmtr_id='status_code'`, not just the generic successful/not_successful step condition, once
  more than one affirmative or negative option exists.
- `task.popup_for_each_row` is the deliberate switch for per-row vs. per-batch confirmation on a bulk
  task — check it explicitly rather than accepting whatever the default happens to be.
- A business-specific database-capture message needs a lower `priority` number than the generic
  base-model capture (commonly `100`) on the same error code, or it will never win.
- `msg` is `type_of_object = 4` and `msg_option` is `type_of_object = 496` for translation purposes —
  use `thinkwise_software_factory_translation_objects`'s generic find-and-overwrite workflow with those
  codes rather than guessing.
- Check usage (`detail_ref_msg_process_action`/`_task`/`_task_variant_overview`) before renaming,
  deleting, or reordering a message's placeholders.
- Don't default a choice message's landing option to the destructive one; reserve type-to-confirm
  friction for genuinely catastrophic, rare actions only — see "UX principles for confirmations and
  choices" above.
