## Editable pivot values

`cube_field.editable` is necessary but **not sufficient** — per the documentation, a pivot cell is
only actually editable when *all* of the following hold simultaneously:

1. The user has Update permission on the underlying subject.
2. The cube field is a value (`measure`), not a dimension.
3. The underlying column itself is editable.
4. `summary_type` is `min`, `max`, `sum`, or `average` (not `count`, `stddev[p]`, `var[p]`,
   `formula`, or `sql_expression`).
5. **Exactly one underlying row** contributes to that aggregated cell — if two rows feed one sum,
   the platform cannot infer how to distribute an edited value back across them. Adding dimensions
   until a cell maps to one row can enable editing, but only when that finer grain is itself
   meaningful and safe to expose.
6. The per-role right is actually granted — `role_cube_field_overview.editable` (distinct from the
   field's own base `editable` flag; both must be true) — see "Permissions" below.

Additional constraints confirmed by the docs: context/default/layout control procedures are **not**
supported on an editable pivot; the after-update refresh behavior should avoid `Document` (it
re-renders the entire pivot on every edit); and editing an analytical result can surprise a user who
expects a cube to be read-only history — label it clearly as planning/input behavior. Use editable
cubes only for genuinely matrix-like planning/allocation where each editable cell backs exactly one
mutable row; otherwise use a Grid, a Formlist, a task, or a dedicated planning table instead.

### Self-referencing hierarchies + editable values — a structural trap

A hierarchy built from a self-referencing column on the *same* table as an editable value (e.g. an
org chart via a `parent_department_id`-style column, with an amount editable per department) has a
failure mode the six preconditions above don't fully protect against. A node with children is not
just a leaf fact in its own right — it is *also* the ancestor bucket that rolls up those children.
When that ancestor bucket happens to aggregate **exactly one** underlying row (a parent with exactly
one child), precondition 5 ("exactly one row per cell") is technically satisfied — the platform allows
editing that cell — but the one row it writes back to is the *child's*, not the row a user looking at
the parent's name would expect to be editing. Verified live: editing this cell silently overwrote the
child's value while appearing to edit the parent's. A parent with two or more children is unaffected
(the aggregate correctly blocks editing there), which makes this specifically a single-child edge
case — easy to miss when testing against a shallow sample hierarchy.

Because `editable` is a property of the `cube_field` itself (not of a `cube_view`), there is no way to
make one view's placement of a field editable and another view's placement of the *same* field
read-only. The fix is to stop sharing the field: add a second `cube_field` pointing at the same
`col_id` with `editable = false` (`cube_field_id` doesn't need to be unique per column — the same
pattern already exists for date-interval hierarchies, where `order_date_year`/`_quarter`/`_month` all
share one `col_id`), use that read-only field's rollup in the hierarchical/browsing view, and keep the
real editable field only in a separate, flat (non-hierarchical) view with no ancestor-bucket ambiguity
at all. Tell users which view is for editing and which is for browsing — the two are not
interchangeable once this split exists.
