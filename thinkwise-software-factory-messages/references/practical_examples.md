# Practical message examples, by pattern family

Worked sketches to draw inspiration from, each naming a real recurring shape and the design decisions
that matter for it. None of this is copy-paste — it's the *contract* (id, severity, location, options,
capture config) a real implementation should match.

## Required business input

- **`email_no_recipients`** — Error, Popup, abort. Text: "The email cannot be sent because no
  recipients are specified. Add at least one recipient and try again." Called from the Handler/Task
  that assembles the email, before any send attempt — not caught as a downstream SMTP failure.
- **`email_no_subject`** — same shape, Warning instead of Error if the model allows sending without a
  subject but wants to flag it; Error if a subject is genuinely mandatory. Deciding which is the actual
  design work here, not the message text.

## Invalid values

- **`negative_numbers_are_not_allowed`** — Error, Popup, abort, parameter `{0}` = the offending value
  via `<text>`. Fired from a Handler/Trigger guarding a quantity column that must stay non-negative
  regardless of write path (UI, API, import) — see "Triggers and handlers" in the main skill for why
  this belongs there rather than only in a Layout.

## Duplicate business keys — layering a business message over the database catch-all

This is the fullest worked example of the "Recommended design" steps in the main skill, using the real
verified base-model catches as the backstop:

1. **Earlier validation** (preferred first line): a Handler or Task checks for the conflicting row
   itself and raises a contextual message *before* the write is attempted —
   **`customer_email_already_registered`** (Error, Popup, abort, `{0}` = the email address). This is
   the message most users actually see.
2. **Database-capture fallback**, for races and any write path that bypasses step 1 (a direct API
   write, a migration script, a concurrent insert): a message targeting the exact unique-index
   violation.
   ```text
   msg_id: index_customer_email_unique
   error_code: 2601
   priority: 10                         -- lower than the generic 100, so this wins
   regex: Cannot insert duplicate key row in object '(?<table>.+)' with unique index
          'ix_customer_email'\. The duplicate key value is \((?<value>.+)\)\.
   text: "A customer with email {value} already exists."
   ```
   Anchoring the literal index name (`ix_customer_email`) inside the regex, rather than leaving it as a
   generic `(?<index>.+)` capture, is what keeps this narrow enough to not accidentally also catch a
   different unique index's violation and mislabel it.
3. **Priority ordering matters**: the base model's own generic `mssql_error_duplicate_key_row` capture
   sits at `priority = 100`. Without step 2's `priority = 10`, the generic message ("Cannot insert
   duplicate key row…") would win and the user would never see the friendlier text — this is exactly
   the "Unstable priority" failure pattern in the main skill's checklist, made concrete.
4. Same pattern for `index_pattern_unique` or any other named unique constraint — one capture message
   per distinct constraint whose violation needs its own wording, each anchored to that constraint's
   exact name and given a priority below the generic catch-all.

## Missing configuration

- **`machine_without_machine_type`** — Error, Popup, abort. Fired from validation before an operation
  that depends on the missing configuration, not surfaced only as a downstream null-reference-shaped
  failure three steps later. Naming the actual missing thing (`machine_type`, not "configuration") is
  what makes the message actionable.

## Business feasibility failures

- **`customer_auto_pack_product_export_not_feasible`** — Error or Warning depending on whether the
  export can be forced through anyway. If forceable, this is a Warning with a **Show message** choice
  ("Export anyway" / "Cancel") rather than a hard Error — see "Wiring into a process flow" in the main
  skill for the status-code routing this needs once there's more than a plain yes/no.

## Successful completion

- **`machine_change_customer_succesful`** — Information, Panel/snackbar, no abort. Deliberately *not* a
  popup — a successful outcome rarely needs to interrupt the user, and panel placement is what keeps
  this from becoming "success noise" (see the failure-patterns list). Reserve a popup-level success
  message for outcomes that genuinely aren't obvious from the resulting screen state (e.g. "3 of 5 rows
  imported, 2 skipped — see the log" needs to be read, a single row save usually doesn't).

## A choice message wired into a process flow

Putting several of the main skill's sections together — a planning run that may overwrite manually
scheduled work:

1. **`msg`**: `planning_run_will_overwrite_manual_scheduling`, Warning, Popup, no abort (the choice
   itself decides whether to proceed, not an automatic abort).
2. **`msg_option`s**:
   - `msg_option_id = 'run_anyway'`, `response_type = true`, `status_code = 0` — deliberately *not*
     reusing the shared `yes` option id, since this message needs its own wording ("Run anyway"), not
     the generic shared "Yes" label.
   - `msg_option_id = 'cancel_run'`, `response_type = false`, `status_code = -1`.
3. **Process flow**: a `show_msg` action (`process_action_type = 350`) with `msg_id` set to the message
   above. Because there are exactly one affirmative and one negative option here, the plain
   `last_process_action_successful = successful/not_successful` step condition on the two outgoing
   `process_step`s is sufficient — the more involved `process_action_modeler_fixed_output`/
   `status_code`/`decision` capture-and-branch pattern in the main skill is only needed once a message
   has *more than one* affirmative or negative option to distinguish between.

## Failure patterns, expanded

- **Popup overuse** — a Default/Layout firing on every keystroke pops a message; the user can't type
  without dismissing dialogs. Fix: move the feedback into field visibility/mandatory state instead of a
  message, or gate the message so it fires once per meaningful change (compare against
  `@cursor_from_col_id`, see `thinkwise_software_factory_subroutines`'s cross-referenced Default
  guidance).
- **Message storms** — a bulk task validates row-by-row and pops one dialog per failing row. Fix: check
  `task.popup_for_each_row` is `false`, aggregate the failures into one summary message with a count and
  a reviewable list, and only then present it once.
- **Wrong severity** — a warning styled message where continuing would actually violate a business
  invariant. Fix: if there's truly no valid way to proceed, it's an Error, not a Warning with an
  "anyway" option.
- **Abort without rollback** — `tsf_send_message` with `abort_ind=1` reports failure to the UI, but the
  preceding `insert`/`update` in the same batch already committed because nothing explicitly rolled it
  back. Fix: always pair an aborting call with an explicit `rollback` (platform permitting) and `return`.
- **Rollback without return** — code rolls back but keeps executing afterward, potentially re-committing
  something else or masking the original failure behind a later, unrelated message. Fix: `return`
  immediately after the rollback.
- **Hand-built XML** — `'<parmtr>' + @customer_name + '</parmtr>'` breaks the moment `@customer_name`
  contains `&`, `<`, or `>` (e.g. "R&D Logistics"). Fix: always `for xml path('text')` or equivalent.
- **Leaked implementation details** — showing the raw constraint name `IX_Customer_Email_Unique` instead
  of "email address" in the user-facing text. Fix: translate the captured identifier into the business
  concept it represents inside the message translation, not inside the regex capture itself.
- **Broad capture regex** — a catch-all pattern on error code `547` (check constraint) matches *every*
  check constraint in the database, so a completely unrelated constraint violation gets mislabeled with
  business text meant for a different one. Fix: anchor the specific constraint name, not just the error
  code, whenever more than one constraint shares that code.
- **Unstable priority** — see the duplicate-key worked example above; the fix is always "give the
  specific message a lower priority number than whatever generic capture already exists on that error
  code."
- **Translation drift** — the English text has `{0}` and `{1}`; a later-added French translation only
  has `{0}`. Fix: check every language contains exactly the placeholders the source uses, no more, no
  fewer, as part of review — not just that a translation exists at all.
- **Vague confirmation** — "Are you sure?" on a delete task. Fix: "Delete order {0}? This cannot be
  undone." names the object and the consequence.
- **Presentation inside reusable logic** — a `subroutine` (see
  `thinkwise_software_factory_subroutines`) that calls `tsf_send_message` unconditionally as part of its
  own body, making it awkward to call from an API or a background job where no popup can ever be seen.
  Fix: return a status/result from the subroutine and let the *caller* (a Task/Handler with a real UI
  context) decide whether and how to present it.
- **Success noise** — a message after every ordinary save when the updated screen already shows the
  result. Fix: default to no message; add one only when the outcome isn't otherwise obvious.
- **Suppression as a fix** — setting `msg_location_id = 'suppress'` on a database error instead of
  fixing the underlying defect that's actually causing it. Fix: suppression is for a message that's
  correct-but-noisy, never a substitute for fixing a real bug.
- **Parsing prose** — an external integration branches its own logic on substrings of the *translated*
  message text. Fix: give integrations a stable machine-readable status/error code to branch on, and
  keep the localized text purely for humans.
