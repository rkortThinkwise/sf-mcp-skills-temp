# Quirks when scripting these changes through a metadata-driven modeling API

These apply broadly across entity types (domains, tables, columns, references, and beyond) whenever
the model is being edited through a metadata/staging-style write API rather than the Software Factory
UI directly — verified live, not assumed:

- **Enum-typed fields can reject the human-readable enum key even though it matches the field's own
  documented enum values, failing with an "invalid input"-style error.** Seen on a domain's default-value-type
  field, a table's table-type field, and a scheduler-style grouping-type field — three unrelated
  entities, same failure shape. If a plain enum key string is rejected, retry the same write using the
  raw underlying numeric value for that enum member instead of its name.
- **Some individual fields can silently fail to persist when set together with other fields in one
  write, with no error returned at all.** Not limited to any one field shape — confirmed across
  description-style long-text fields (committing successfully but not actually saving the text),
  sequence/order-number fields (independently on a domain-element-style child row's own field and on a
  menu item's `order_no`), boolean flags (a task's confirmation-prompt toggle, a table-task's
  enable-when-no-row-selected toggle), and plain string/identifier fields (a look-up column-mapping's
  target parameter id, a menu item's target-table id). Seen across at least five different entity types
  spanning task, table-task, task-look-up, control-procedure, and menu objects in one session — this is
  general write-API behavior, not a quirk of order-number or description fields specifically. Treat
  **any** field as suspect after a combined write, not just those two shapes. A separate session
  building over 20 records through this same kind of write flow saw the drop land specifically on the
  **last property in the write's ordered list, every single time** — never a first or middle one —
  which makes the last field the single highest-value one to check (or just always re-send) first,
  even though in principle any field can be affected. After any multi-field write to a record, re-read
  back the fields that matter before moving on, and if one didn't take, retry it alone in an isolated
  single-field write — when scripting many records in sequence, patching one field per call (verifying
  each response) is the safer default over batching several fields per call, even though batching
  usually works.
- **A table's child rows can carry their own separate NOT-NULL order-number field(s), distinct from
  and in addition to whatever `order_no` the row already has, with no database default.** Verified
  live on `col`: beyond its own `order_no`, a newly-inserted column also has `filter_order_no`,
  `search_order_no`, `grid_order_no`, `form_order_no`, and `card_list_order_no` — a hand-written
  `INSERT` that only populates `order_no` fails with a null-constraint error naming the friendly
  field (e.g. "Filter order number"), not the raw column name. The same shape hit `ref_col.order_no`
  ("Sequence no") — a reference's own child join-column rows need their position-derived `order_no`
  supplied explicitly too, it is not backfilled automatically. Before writing or reviewing any raw
  `INSERT`/`MERGE` against `col`, `ref_col`, or similar position-keyed child tables, check the target
  entity's full field list for additional NOT-NULL order-number fields beyond the one already in the
  insert list.
- **Requesting several large, property-heavy entity types' full metadata in one introspection call can
  overflow the response size limit.** Table-, column-, and reference-style entities in particular carry
  many properties, navigation properties, and bound actions — request at most one or two such entity
  types per introspection call rather than batching many together, especially early in a session
  before it's clear how large a given entity's schema is.
- **A weak entity's usual parent-based "add" can fail to resolve its parent even though the parent
  object genuinely exists** — seen when the parent lives outside the child's normal navigation context
  (e.g. a task's own parameter row, whose real parent is the task object itself rather than whatever
  table the task happens to be attached to). If a parent-based add returns a "parent not found"-style
  error for an object you've confirmed exists, fall back to a **plain add**, supplying the full
  compound key (including the columns that would normally come from the parent) as ordinary fields
  directly — this succeeds where the parent-nav path doesn't.
- **A description-style translatable field can appear as a normal editable field on a freshly-staged
  new record, yet a direct write to it is rejected outright** (an "unknown property"-style error, not a
  silent no-op) — unconfirmed root cause, but consistent with such fields routing through the
  translation-object mechanism rather than a plain column patch, at least before the record's first
  commit. If a description-style field's write is rejected this way, skip it during initial creation
  and set it afterward via `thinkwise_software_factory_translation_objects` instead of retrying the
  same patch.
- **Never trust an unfiltered query to confirm an object exists in the model you're working in.** Many
  entity sets (e.g. `screen_type`) are not scoped to "the current model" by default — a query with no
  explicit `model_id`/`branch_id` filter can match rows belonging to a completely different model or
  branch in the same shared repository. A match under a bare `contains(...)`-only filter is not proof
  the object is usable in your model; always re-run the check with `model_id`/`branch_id` filters added
  before relying on the result — e.g. before assigning a system-provided screen type by name.
- **Creating a new `screen_type` (and placing components/leaf panels onto it) is not currently
  possible through this kind of write API.** Treat screen type creation as a manual Software Factory
  UI step, and always plan around reusing a screen type that already exists in the model rather than
  assuming one can be created on demand. See `thinkwise_software_factory_build_planner`'s
  interaction-surface step for how to choose among existing screen types — including asking the user
  what the screen should look like, and flagging a manual-creation prerequisite when nothing existing
  fits.
- **A blanket rejection on writes to an entity that should be editable can be a Software Factory
  role/rights gap on the current session's account, not a structural API limitation.** Confirmed live:
  an entity family that consistently rejected every add/edit attempt across several different approaches
  started working normally, with zero code or model changes, purely after the user adjusted the
  connected account's Software Factory roles/rights. Before concluding (and documenting) that some entity/task is
  unsupported through this kind of write API, consider asking the user to check their Software Factory
  role/rights configuration and retry — it can silently explain a failure that otherwise looks
  identical to a genuine connector/platform limitation.
  **The same gap can hide a task's existence entirely, not just block writes to an already-visible
  entity — confirmed live**: a bound task was completely absent from its entity's bound-task list, and
  a direct lookup by its exact name returned a hard "not found," indistinguishable from the task
  genuinely not existing on that entity — until the account's role/rights were adjusted, at which
  point the identical introspection call showed the task and flipped the entity's own
  add/update/delete permissions from all-denied to all-allowed. A task or entity that looks entirely
  absent from metadata, not just write-rejected, is still worth a role/rights check before concluding
  it isn't there.
