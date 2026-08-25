# Writing SQL in the right dialect

**Query `branch_rdbms_type` first (see the top of the main SKILL.md) — don't assume, don't default to
T-SQL out of habit.** Four target platforms: SQL Server (T-SQL, `rdbms_type` 0), DB2 iSeries (1),
Oracle (3), PostgreSQL (4). Thinkwise's own SQL guidelines are T-SQL-specific; no official
cross-dialect table exists, which makes it easy to reach for T-SQL by default even against a model
that doesn't run it.

**Use Thinkwise's own dialect-safe helpers instead of hand-rolling platform calls:**
- **Current user** — `tsf_user()`, call syntax differs per platform: `select dbo.tsf_user();` (SQL
  Server), `select tsf_user() from dual;` (Oracle), `select tsf_user() from sysibm.sysdummy1;` (DB2
  iSeries), `select public.tsf_user();` (PostgreSQL). Under simulation, `tsf_user()` returns the
  *simulated* user; `tsf_original_login()` always returns the real user (needs Indicium/session
  context). Neither works in Crystal Reports/Generate report — pass username in as a parameter via a
  Default calling `tsf_user()` instead.
- **Messages/errors** — `tsf_send_message [message_id], [parameter_xml], [abort]`, identical call
  shape everywhere, but `@abort = 1` behaves differently: on **SQL Server** it does not itself stop
  execution — still write `rollback; return;` yourself. On **Oracle** it triggers
  `raise_application_error` internally, which already unwinds as a propagating exception — no
  manual rollback needed (`@abort = 0` instead calls `dbms_output.put_line`). Use
  `tsf_send_message` instead of `raiserror` always. Layout/Context/Process **must never** send
  messages at all (see `references/sql_style_guide.md`, "By code type"); Default/Trigger-event/
  Handler/Task/Subroutine can.
- **Identity retrieval in a Handler** — SQL Server: `SCOPE_IDENTITY()`. Oracle: can't mutate an `in`
  parameter directly, so use a sequence + mutable `m_`-prefixed local + `returning … into`.

**Where no Thinkwise helper exists**, prefer the ANSI/portable form when every target platform
supports it (fork by `rdbms_type` when they don't) — but see the style-continuity rule in
`references/sql_style_guide.md` before introducing a new one into an existing codebase:

| Need | SQL Server | Oracle | DB2 iSeries | PostgreSQL | Portable choice |
|---|---|---|---|---|---|
| Null fallback | `isnull(x,y)` | `nvl(x,y)` | `coalesce(x,y)` | `coalesce(x,y)` | `coalesce(x,y)` |
| Current timestamp | `getdate()`/`sysutcdatetime()` | `sysdate` | `current timestamp` | `now()`/`current_timestamp` | `current_timestamp` |
| String concat | `+` or `concat()` | `\|\|` | `\|\|` | `\|\|` or `concat()`¹ | `concat()` |
| Limit row count | `top (n)` | `fetch first n rows only` | `fetch first n rows only` | `limit n` | `fetch first n rows only` |

¹ Not inside a PostgreSQL generated/persisted `calculated_column` expression — see below.

Push anything genuinely platform-exotic (linked servers, `xp_` procs, DB2 job-log queries, Oracle
`dbms_*` packages) into a Task or Subroutine — Layout/Context/Process can't touch tables or send
messages anyway. Test generation and execution on every platform the model actually targets before
deploying.

## PostgreSQL generated/calculated columns require IMMUTABLE functions

**Verified live (`42P17`)**: PostgreSQL rejects a native generated column (`GENERATED ALWAYS AS
(<expr>) STORED` — what a Thinkwise `calculated_column` + `PERSISTED` maps to) whose expression
calls a function not marked `IMMUTABLE`. This is PostgreSQL-specific; SQL Server computed columns
have no equivalent restriction.

**Cause of the specific bug hit live**: `concat()` is marked `STABLE`, not `IMMUTABLE`, in
PostgreSQL, because it accepts `anyelement` arguments and calls each type's output function
internally — it looks like a pure string function but isn't classified as one.

**Fix**: use `||` instead of `concat()` inside a `calculated_column`/`PERSISTED` expression — `||`
is immutable for ordinary text-backed domains. **Behavioral difference to flag, not silently
absorb**: `concat()` treats a `NULL` argument as an empty string; `||` propagates `NULL` (any
`NULL` operand nulls the whole result). If the original expression relied on `concat()`'s
NULL-tolerance, wrap each operand instead of a bare operator swap:
`coalesce(x, '') || coalesce(y, '')`.

**Doesn't apply to `expression`-type calculated columns** (a correlated subquery re-evaluated per
query, not a stored generated column) — `concat()` is fine there. Only a `calculated_column`
(native generated/persisted column) hits this.

Don't assume a function "looks pure" ⇒ `IMMUTABLE` — check PostgreSQL's own volatility
classification (`\df+ <funcname>` or the docs) before using it inside a generated column or a
functional index, the two contexts where this actually matters.
