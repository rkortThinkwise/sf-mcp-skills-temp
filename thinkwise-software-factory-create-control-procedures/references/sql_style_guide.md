# Thinkwise SQL coding guidelines

**0 · Match the codebase's existing style — the most important rule, not an official Thinkwise
one.** Before writing a trace timestamp, a null-check, a string concatenation, or a user lookup,
check how the model already does it and copy that — don't introduce a second convention next to an
existing one, even if the new one is technically equivalent or "more correct" in isolation. Verified
example from a real production model: it standardized on `sysutcdatetime()` (never `getdate()`/
`getutcdate()`), `isnull()` (never `coalesce()`), and `dbo.tsf_user()` everywhere a current-user
lookup was needed. Introducing the portable/ANSI alternative from the dialect table in
`references/sql_dialects.md` into that codebase would be correct in isolation and wrong for that
codebase. Consistency beats being technically right. Grep the model's existing control procedures
first, every time.

**General:**
- Use a domain with domain elements if you need to program on values (not magic strings/numbers).
- Avoid `distinct` (e.g. count via a `group by` in a derived table instead of `count(distinct …)`).
- Avoid `union` without `all`.
- Use `local static read_only forward_only` cursors when a cursor is genuinely unavoidable.
- Use `begin`/`end` in every `if`/`while`, even single-line ones.
- Use `tsf_send_message` instead of `raiserror`, always.
- Only use `apply` when joining a table-valued function or subquery.
- Every `begin tran` needs a matching `rollback tran` and `commit tran`.
- **A recursive CTE can't be nested inside any subquery expression, in any code type — verified
  live.** This is broader than a Trigger-specific restriction: `if exists (with cte as (...) select
  1 from cte ...)` fails with a syntax error near `with` on a Handler exactly the same way it does on
  a Trigger, even though the CTE is immediately followed by the single query that consumes it — the
  CTE has to stand as its own statement, not live inside an `exists()`/`in()`/scalar subquery. For a
  code type that can use variables (Handler/Task/Subroutine — see the per-code-type restrictions
  below for which ones can't), run the CTE into a flag variable first as its own statement, then
  branch on the variable: `declare @flag bit = 0` / `select @flag = 1 from cte where ...` / `if @flag
  = 1 begin ... end`. Where variables aren't allowed (Triggers), see that section's bounded self-join
  workaround instead — it doesn't need a variable at all.

**By code type:**
- **Triggers**: functional-integrity validation, not complex data mutation (use a Handler/Task for
  that). Every statement should reference `inserted`/`deleted`. No cursors, no explicit
  transactions, no variables, no temp tables, never modify `inserted`/`deleted` directly.
  **A recursive CTE can't be used to feed an `if` check here, verified live**: `with cte as (...)
  if exists (select ... from cte ...)` is invalid T-SQL — a CTE must be immediately followed by the
  single query that consumes it, not a control-flow statement. This surfaces only as a deploy-time
  syntax error ("Incorrect syntax near the keyword 'if'"), not a generation-time failure, so a
  "Successful" code-generation status doesn't catch it. For a hierarchical/recursive check inside a
  Trigger — where the restrictions above also rule out capturing the CTE's result in a variable or
  temp table first — use a bounded chain of self-joins walking a fixed number of levels instead
  (e.g. 15-20 `left join`s up a self-referencing parent column): fully set-based, no CTE, variable,
  or temp table required.
- **Defaults**: wrap in `if`; don't reassign an input parameter's incoming value blindly. No
  cursors, no explicit transactions, no table writes, no temp tables.
- **Layouts, Contexts, Processes**: same restrictions as Defaults, **plus**: never send messages
  here at all.
- **Tasks & Subroutines (stored procedures)**: avoid cursors where possible; always wrap in
  explicit transactions — **verified live: for Task and Handler code types, the code group's own
  generated wrapper items (e.g. a `transaction_start`/`transaction_commit`/`transaction_catch` set)
  already bookend your assigned template as separate `prog_object_item` rows, so your own template
  should contain just the business logic, not its own `begin tran`/`commit tran`; check
  `prog_object_item` for the target object before adding transaction statements by hand**; if a cursor
  is unavoidable, wrap each iteration in its own transaction.
- **Subroutines (functions)**: no cursors, no explicit transactions, no table writes, no messaging —
  pure functions only.

**Formatting:** spaces not tabs; lowercase everywhere (keywords, built-in functions, data types,
session variables); comment every code block and non-obvious literal; strip dev-only helper code
before shipping; explicit column list on every `insert`; alias calculated values/literals and align
aliases; commas before the column name; align comparison operators.

## Efficient Trigger example — avoid the cursor instinct

A trigger fires once per statement, not once per row — a multi-row insert lands in `inserted` all at
once. Looping through it row-by-row with a cursor throws away exactly what makes triggers fast, and
violates every Trigger rule above simultaneously:

```sql
-- Anti-pattern — do not do this in a trigger
declare @inserted_rows table (id int, value varchar(max))

insert into @inserted_rows (id, value)
select order_id, order_number
from inserted

declare @id int
declare cursor_id cursor local static read_only forward_only for
select id
from @inserted_rows

open cursor_id
fetch next from cursor_id into @id

while @@fetch_status = 0
begin
    insert into order_insert_log (order_id, log_message)
    values (@id, 'order inserted')

    fetch next from cursor_id into @id
end

close cursor_id
deallocate cursor_id
```

```sql
-- Efficient rewrite — set-based, no cursor, references inserted directly
insert into order_insert_log (order_id, log_message)
select order_id, 'order inserted'
from inserted
```

One statement, no cursor, no variables, no temp table — the entire batch logs in one set-based
operation whether the triggering statement affected 1 row or 10,000. The `local static read_only
forward_only` cursor exception in the General guidelines is for Tasks/Subroutines, not Triggers —
needing a cursor inside a trigger is usually a sign the row-by-row work belongs in a Handler or Task
instead.
