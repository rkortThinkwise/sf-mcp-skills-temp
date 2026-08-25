# Recursive/hierarchical (explosion) views

For a self-referencing hierarchy (a bill-of-materials explosion, an org chart, a category tree), the
Template `SELECT` is almost always a recursive CTE. Two SQL Server-specific gotchas, both verified
live, hit on the very first recursive view most sessions write:

- **Never put `OPTION (MAXRECURSION n)` inside the view's `SELECT`.** SQL Server rejects a query hint
  embedded in a view's defining statement with a syntax error at deploy time ("Incorrect syntax near
  the keyword 'option'") — hints only attach to a query executed directly, never to a stored view
  definition. A clean code-generation status doesn't catch this; it only surfaces once the generated
  `CREATE VIEW` is actually deployed. If the recursion genuinely needs to go deeper than SQL Server's
  default limit of 100, that's not achievable from inside a view at all — wrap the query in a
  Subroutine/Task instead, where a hint on the query issuing the call can apply.
- **Cast every domain-backed column referenced in the recursive member back to the anchor's exact
  base type — not just literals.** Thinkwise generates each domain as its own SQL Server user-defined
  type alias (e.g. a `quantity` column typed as an alias over `numeric(18,3)`, not a bare
  `numeric(18,3)`). SQL Server's recursive CTE requires the anchor and recursive members' column
  types to match *exactly*; an aliased-type column and a literal cast to the same underlying base
  type (e.g. `cast(1 as decimal(18,3))` in the anchor) are rejected as a mismatch ("Types don't match
  between the anchor and the recursive part in column ..."), even though they're identical
  underneath. Explicitly `cast()` the aliased column in the recursive member to the same base type
  the anchor uses.
- **Re-cast accumulating arithmetic back to the fixed target type on every recursive step, not just
  once.** A running total/product (e.g. `t.cumulative_quantity * pb.quantity`) computed across
  recursion levels grows precision/scale under SQL Server's own decimal-arithmetic rules
  (`decimal(18,3) * decimal(18,3)` → `decimal(37,6)`, not `decimal(18,3)`) — left uncast, this trips
  the identical anchor/recursive type-mismatch error on that column right after fixing the one above.
  Wrap the whole expression in an explicit `cast(... as <target type>)` matching the anchor's cast.
