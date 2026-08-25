# Business-logic variables by code type

`[col_id]`/`[ref_id]`/`[task_id]` mean one such variable per matching column/tab/task, named after
the object (e.g. `@activated_type` for a Layout on column `activated`).

| Code type | Input | Output / settable |
|---|---|---|
| Default | `@default_mode` (0 insert/1 update), `@import_mode` (0 UI/1 API/3 import), `@cursor_from_col_id` (null on add), `@[col_id]` | `@[col_id]`, `@cursor_to_col_id`, `@auto_commit` |
| Layout | `@layout_mode` (0/1/2 insert/update/navigate), `@import_mode` (0/1/2/3 UI/API/export/import), `@[col_id]` | `@[col_id]_type` (0 normal/1 read-only/2 hidden in form/3 hidden outside form), `@[col_id]_mand`, `@add_button_type`, `@update_button_type`, `@delete_button_type`, `@confirm_button_type`, `@cancel_button_type` |
| Context | `@active_ref_id`, `@[col_id]` | `@[ref_id]_type`, `@[task_id]_type`, `@[report_id]_type` (0 enabled/1 disabled/2 hidden) |
| Trigger/event | create/update/delete rows in `inserted`/`deleted` pseudo-tables (old values on update/delete, new on create/update) | — (before/after/instead-of; can roll back the transaction) |
| Handler | Insert: all columns; Update: all columns + PK; Delete: PK only | Insert: identity via `SCOPE_IDENTITY()` (Oracle/DB2: mutable `m_`-prefixed locals mirroring `in out` params, since those platforms disallow mutating `in` parameters) |
| Change detection | `@variant_id`, `@last_refresh_utc` (null if never refreshed), `@[col_id]` (detail-mapped columns only) | `@refresh_data` (0/1, default 0) |
| Badge | `@variant_id`, `@[col_id]` (column/linked task or report parameter that's part of a detail FK) | `@badge_value` (integer; `null` hides the badge) |
| Process | `@[process_variable_id]` (flow variables marked Process input/output available) | `@[follow_up_process_action_id]` — null/0/negative skips that action |
| Task | Same Default/Layout mechanism as tables, but `@[col_id]` is a task parameter, not a column | Same as Default/Layout; Layout only affects form styling (tasks have no grid) |

**Handler PK parameter on Update: two different values, easy to swap by mistake.** The "all columns +
PK" input for an Update handler splits into `@upd_[pk_col_id]` — the row's **current** key value, what
the `where` clause should filter on — and the plain `@[pk_col_id]` — the **new** value being written
(relevant only if the PK is nominally editable; for an immutable PK the two carry the same value, but
they are still two distinct parameters). Locate the row with `@upd_[pk_col_id]`, not the bare one.

## `@cursor_from_col_id` — initial default vs. reactive recompute

For the Default code type, `@cursor_from_col_id` distinguishes two different moments the same
procedure fires at: it's empty the first time defaults run (a new row, or a task's input form just
opening — nothing has been edited yet), and holds the id of whichever column/parameter the user just
edited on every later re-evaluation triggered by that edit. What "empty" means can vary by platform/
dialect — check for both `is null` and `= ''` rather than assuming one.

Use it to pick the behavior actually wanted:
- **Compute once, then respect manual edits** — guard the whole body on `@cursor_from_col_id` being
  empty. The value is suggested once (e.g. when the form opens) and never silently overwritten again,
  even if the input it was derived from changes later. Verified live (Task Default, T-SQL):
  ```sql
  if @start_date_time is not null and (@cursor_from_col_id is null or @cursor_from_col_id = '')
  begin
      set @end_date_time = dateadd(day, 2, @start_date_time)
  end
  ```
  End date defaults from start date only on the initial pass; once the user has touched the form, a
  later edit to start date no longer resets a manually-adjusted end date.
- **Recompute every time a specific field changes** — check `@cursor_from_col_id = '<driving column
  id>'` instead (or omit the check entirely to recompute on every pass). Use this when the dependent
  value should always track its source, even after the user has interacted with the form.

Omitting the guard when "once, then leave it alone" was actually wanted is what makes a Default feel
like it's fighting the user — the value keeps getting reset on unrelated field changes.

**Variable name prefix is dialect-dependent.** The table above uses the T-SQL `@[col_id]` form.
Verified on a real PostgreSQL model: the same variables are named `p_[col_id]` (e.g. `p_start_date`,
`p_default_mode`), and assignment uses `:=` instead of `set … =` (`p_release_date := current_date;`
not `set @release_date = current_date`). Don't assume `@`-prefixed T-SQL syntax carries over —
confirm the model's RDBMS (control procedure IDs prefixed `pg_*` are a quick tell for PostgreSQL) and
grep an existing template in the same code type for the real prefix/assignment style before writing
new code.

## Getting the current user

`tsf_user()` (session login) and `tsf_original_login()` (the underlying login when impersonation/pool
accounts are involved) return the acting user inside a control procedure. **Crystal reports, and the
*Generate report* process action, run as the pool user instead of the actual end user** — if a report's
logic needs the real user, don't call `tsf_user()` from the report itself; pass the user in as an input
parameter, filled by a Default that calls `tsf_user()` in a context where it still resolves correctly.
