# T-SQL → PostgreSQL conversion reference

Everything here was hit and verified live while porting a real model's hand-written SQL (control
procedures, calculated fields, prefilters). This complements — doesn't replace —
`thinkwise_software_factory_create_control_procedures`'s `references/sql_dialects.md`, which covers
all four platforms in general; this file is PostgreSQL-migration-specific and includes the structural
rewrites that table doesn't attempt to cover.

## Mechanical substitutions (find, don't think)

| T-SQL | PostgreSQL | Notes |
|---|---|---|
| `@[name]` business-logic variables | `p_[name]` | Confirmed dialect-dependent prefix |
| `set @x = y` | `x := y` | PL/pgSQL assignment |
| `isnull(x, y)` | `coalesce(x, y)` | |
| `getdate()` | `current_date` or `current_timestamp` | Pick based on the column's actual domain (date vs. datetime) — using the wrong one compiles fine and is wrong |
| `sysdatetime()` | `current_timestamp` | |
| `nvarchar(n)`, `varchar(n)` | `varchar(n)` | No `nvarchar` in PostgreSQL |
| `nvarchar(max)` | `text` | |
| `datetime2` | `timestamp` | |
| `+` string concat | `\|\|` or `concat()` | `concat()` behaves the same both platforms (NULL args become empty string) — **except** inside a `calculated_column`/`PERSISTED` expression, where PostgreSQL requires `\|\|` instead; see "PostgreSQL generated columns require IMMUTABLE functions" below |
| `charindex(x, y)` | `position(x in y)` | |
| `substring(x, start, len)` | `substring(x, start, len)` | PostgreSQL supports the same positional 3-arg form directly — no change needed |
| `len(x)` | `length(x)` | |
| `left(x, n)` | `left(x, n)` | PostgreSQL has the same function — no change needed |
| `top n` (as a query prefix) | `limit n` (as a query suffix) | Also works inside a scalar subquery: `(select x from t order by y desc limit 1)` |
| `top n` combined with `order by` desc/asc | `order by ... limit n` | Same reordering as above |
| `1`/`0` literal against a BIT-backed (now boolean) column | `true`/`false` | PostgreSQL booleans don't implicitly cast from integer — `col = 1` on a real boolean column is a type error, not a silent truthy check |
| `convert(int, 0xHEXVALUE)` | `x'HEXVALUE'::int` | Hex literal syntax differs |
| `case when x = 1 then ... end` (x is boolean-backed) | `case when x then ...` / `case when x = true then ...` | Same boolean-literal issue as above, inside a `case` |
| `datefromparts(y, m, d)` | `make_date(y, m, d)` or `date_trunc('month', current_date)::date` for month-start | Pick whichever matches the actual intent — most real uses are "start of current month," better expressed with `date_trunc` |
| `dateadd(month, n, x)` | `x + interval 'n months'` (or `(x::date + (n || ' months')::interval)::date` if `x` isn't already a date) | Interval arithmetic on a `date` returns `timestamp` in PostgreSQL — cast back to `date` if the target column needs one |
| `dateadd(day, n, x)` | `x + interval 'n days'`, or plain `x + n` if `x` is a `date` (integer days arithmetic works directly on `date`) | Prefer the plain integer form on a `date` column — simpler and avoids the timestamp-cast issue above |
| `N'literal'` (nvarchar string prefix) | `'literal'` | No `N` prefix needed |
| `try_cast(x as t)` | No direct equivalent | See "No `try_cast`" below |
| `json_value(x, '$.Path')` | `(x::jsonb ->> 'Path')` | Cast to `jsonb` first, then extract |
| `scope_identity()` (Handler insert) | `insert into t (...) values (...) returning pk_col into v_var` | See "Identity retrieval" below |
| `exec proc_name @p1 = @v1` | `call proc_name(p_p1 => v_v1)` | PostgreSQL uses `call` for procedures, `=>` for named parameters |
| declare-a-cursor / open / fetch loop / close / deallocate | `for v_row in (select ...) loop ... end loop;` | PL/pgSQL's `for ... in (query) loop` replaces the entire T-SQL cursor boilerplate — a real simplification opportunity, not just a syntax swap |
| `with mycte as (...) select ... from mycte` (recursive) | `with recursive mycte as (...) select ...` | PostgreSQL requires the literal keyword `recursive`; T-SQL doesn't |
| `option (maxrecursion n)` | No direct equivalent | Bound recursion explicitly instead — add a depth column to the recursive CTE and `where depth < n` in the recursive term |
| SQL Server computed-column `... PERSISTED` suffix (in a `calculated_field_query` value) | Drop it entirely | PostgreSQL has no equivalent keyword at this level; verify after generation whether the framework still produces a stored/materialized column without it — unconfirmed in the reference migration |

## PostgreSQL generated columns require IMMUTABLE functions — confirmed via 42P17

**Symptom**: generating/validating a `calculated_column` (persisted) expression on PostgreSQL
fails with `42P17`, "generation expression is not immutable."

**Cause**: the function used inside `col_query.calculated_field_query` is `STABLE` or `VOLATILE`,
not `IMMUTABLE` — PostgreSQL requires every function in a `GENERATED ALWAYS AS (...) STORED`
expression to be immutable. Confirmed live: `concat()` is `STABLE` (it dispatches to each
argument's type output function internally).

**Fix**: replace `concat()` with `||` — immutable for ordinary text domains. If the original
expression relied on `concat()`'s NULL-to-empty-string behavior, wrap each operand instead of a
bare operator swap: `coalesce(x, '') || coalesce(y, '')`. See
`thinkwise_software_factory_create_control_procedures`'s `references/sql_dialects.md` for the
general rule — check this for *any* function used in a `calculated_column` expression being
ported, not just `concat()`.

## No `try_cast` — write an exception-guarded cast instead

T-SQL's `try_cast` returns `NULL` on a failed conversion instead of erroring. PostgreSQL has no
equivalent expression. If the original logic depends on that null-on-failure behavior (not just doing
a cast that's always known to succeed), wrap it in a small `plpgsql` block:

```sql
declare
    v_result numeric(10,2);
begin
    begin
        v_result := (some_text_value::jsonb ->> 'Radius')::numeric(10,2);
    exception when others then
        v_result := null;
    end;
    -- use v_result
end;
```

## Identity retrieval in a Handler insert

T-SQL:
```sql
set @new_id = cast(scope_identity() as int);
```

PostgreSQL — use `returning ... into` directly on the insert instead of a separate lookup:
```sql
insert into account (account_name, ...) values (...)
returning account_id into v_new_account_id;
```

## `tsf_send_message` abort semantics differ by platform — this changes call sites, not just the function

Verified live by generating the framework's own PostgreSQL implementation: on **PostgreSQL**,
`tsf_send_message(..., abort => true)` raises a real PostgreSQL exception (`raise exception`)
immediately — execution does not continue past that line, and the enclosing transaction unwinds
automatically. On **SQL Server**, `exec tsf_send_message ..., 1` does *not* itself stop execution —
existing T-SQL templates always follow it with an explicit `if @@trancount > 0 rollback transaction`
and `return`.

**When porting a template that calls this**: drop the `if @@trancount > 0 rollback transaction` /
`return` entirely — it becomes dead code after the exception-raising call, and leaving it in doesn't
break anything but is misleading about what actually happens. This mirrors the already-documented
Oracle behavior in `sql_dialects.md`'s helper-function table (Oracle's `abort` also unwinds via a real
exception) — PostgreSQL behaves the same way here, unlike SQL Server.

```sql
-- T-SQL original
if @budget_amount < 0
begin
    exec tsf_send_message 'default', '<text>...</text>', 1
    if @@trancount > 0
        rollback transaction
    return
end

-- PostgreSQL port
if p_budget_amount < 0 then
    call tsf_send_message('default', '<text>...</text>', true);
end if;
```

## Cross-item shared state doesn't survive the port — plpgsql can't `declare` after `begin`

**The pattern this breaks**: in T-SQL, several `control_proc_template` items assigned to the same
`prog_object_id` concatenate into one generated body in `order_no` order, and a local variable
`declare`d in an early (pre-mutation) item is visible to a later (post-mutation) item in the same
object — a documented, legitimate way to carry a captured "before" value across the mutation
(see `thinkwise_software_factory_create_control_procedures`'s note on this).

**Why it breaks on PostgreSQL**: the framework's own PostgreSQL wrapper item (`handler_start`) ends
with a bare `begin` — there is no top-level `declare` section for custom items to extend, and PL/pgSQL
syntactically requires `declare` to precede `begin`, not follow it. A variable declared inside one
item's own nested `declare ... begin ... end;` block (the correct way to have local variables at all in
a PostgreSQL item) is scoped to that block alone and invisible to a sibling item later in the same
object.

**Fix, verified live**: use a transaction-scoped session setting to carry the value across, mirroring
the same mechanism Thinkwise's own framework uses for session context (`tsf_appl_id` and friends use
`current_setting`/`SESSION_CONTEXT` the same way):

```sql
-- pre-mutation item: capture the old value before the framework's own update statement runs
declare
    v_old_cost decimal(18,3);
begin
    select cost into v_old_cost from product where product_id = p_upd_product_id;
    perform set_config('session.product_cost_guard_old_cost', v_old_cost::text, true);
    -- ... validation using v_old_cost ...
end;

-- (the framework's generated UPDATE statement runs here, between the two items)

-- post-mutation item: read it back
declare
    v_old_cost decimal(18,3) := nullif(current_setting('session.product_cost_guard_old_cost', true), '')::decimal(18,3);
begin
    -- ... cascade logic using v_old_cost ...
end;
```

`set_config(..., true)` (the third argument) scopes the setting to the current transaction — it clears
automatically, matching the lifetime a T-SQL local variable would have had. Namespace the setting key
uniquely per use (`session.<control_proc_id>_<purpose>`) to avoid collisions with unrelated logic in
the same session.

**Any handler input variable that's independently available in both items doesn't need this at all**
— only a genuinely "before the mutation" value that isn't otherwise derivable needs the session-setting
carry. Re-deriving a value from a plain input parameter (e.g. re-joining through `p_product_type_id`,
which is available unchanged in both the pre- and post-mutation item) is simpler and doesn't need any
of this.

## Triggers: batch `inserted`/`deleted` (T-SQL) → row-level `NEW`/`OLD` (PostgreSQL) — a redesign, not a port

T-SQL AFTER triggers see the whole batch of affected rows via the `inserted`/`deleted` pseudo-tables,
which is why some T-SQL trigger code resorts to awkward workarounds — e.g. a fixed-depth chain of
`left join`s to walk a hierarchy, specifically because Trigger code in T-SQL disallows a CTE followed
by anything other than the consuming query.

PostgreSQL triggers fire per-row (`for each row`) against `NEW`/`OLD` record variables, and a
`plpgsql` trigger function can use an ordinary (recursive) CTE freely — the T-SQL-specific restriction
that motivated the workaround doesn't exist here. **Don't port the workaround verbatim** — take the
opportunity to write the natural version:

```sql
-- T-SQL original: 15-level join chain against `inserted`, to prevent circular parentage
if exists (
    select 1
    from inserted i
    inner join department d1 on d1.department_id = i.parent_department_id
    left join department d2 on d2.department_id = d1.parent_department_id
    -- ... 13 more levels ...
    where i.parent_department_id is not null
      and i.department_id in (d1.department_id, d2.department_id, /* ... */)
)
begin
    exec tsf_send_message 'default', '<text>...</text>', 1
    if @@trancount > 0 rollback transaction
    return
end

-- PostgreSQL: a proper unbounded (but depth-capped for safety) recursive CTE against NEW,
-- as a BEFORE trigger so the check runs before the write instead of rolling back after it
if new.parent_department_id is not null then
    if exists (
        with recursive ancestors(department_id, parent_department_id, depth) as (
            select d.department_id, d.parent_department_id, 1
            from department d
            where d.department_id = new.parent_department_id
            union all
            select d.department_id, d.parent_department_id, a.depth + 1
            from department d
            join ancestors a on d.department_id = a.parent_department_id
            where a.depth < 100
        )
        select 1 from ancestors where department_id = new.department_id
    ) then
        call tsf_send_message('default', '<text>...</text>', true);
    end if;
end if;
```

Note also: prefer `before` over `after` for a pure validation/rejection trigger on PostgreSQL — it
lets the exception stop the write outright, instead of the T-SQL pattern of letting the write happen
and then rolling it back. The framework generates four distinct trigger flavors on PostgreSQL where
T-SQL had one undifferentiated trigger object — `pg_before_triggers_row`, `pg_after_triggers_row`,
`pg_before_triggers_statement`, `pg_after_triggers_statement` (naming convention:
`<table>_brt<type>`/`<table>_art<type>` for row-level, `<table>_bds`/`<table>_ads`-style for
statement-level) — pick the matching one deliberately; it's a real design choice, not a naming detail.
