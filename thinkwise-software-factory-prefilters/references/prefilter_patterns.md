## Common patterns

**1 · Status checklist off a domain element** (column-based) — `declaration_lines.status`, domain
`declaration_status` → one prefilter per value the users actually filter by (`approved`, `cancelled`,
`paid`, `to_be_approved`), each `Column = status`, `Condition = Equal to`, `Value = <element's database
value>` (the stored code, e.g. `2` — not the element's ID/name `approved` and not its translated
caption), grouped so they render as one set of chips. The general pattern: any column typed to a domain with a
fixed value set (status, category, type) is a candidate — model one prefilter per value that's
actually useful to filter by, not necessarily every value the domain defines. Reuse the domain
guidance in `thinkwise_datamodeling_guidelines` when deciding whether the underlying column even needs
a new domain versus reusing an existing one.

**Caution**: a column's check constraint (e.g. `"status" in (0, 1, 2, 3)`) confirms only the valid
*range* of codes, never their captions — don't infer a value's meaning from its position in the range.
Real captions are typically modeled as translations, in a domain separate from the table/column
metadata itself, and aren't necessarily reachable as simple per-value rows through it. Look for a
corroborating source before asserting a label — a view or template's own SQL (a `case` mapping the
code to a color or display string, for instance) is often available and far more reliable than
guessing. If no such source exists, present the codes with plainly-flagged best-guess captions rather
than asserting them as confirmed.

**2 · Exclusive two-state toggle** (column-based) — `email.is_read`, domain `no_yes` → `read` (Equal
to, Yes) and `unread` (Equal to, No) in a group with `allow_multiple_active_prefilters = false`. That
setting alone turns two independent toggles into one exclusive switch. Use for any genuinely binary,
mutually-exclusive column — read flags, active/archived, in-stock/out-of-stock.

**3 · "My records"** (query-based) — an `exists` subquery against a per-user settings table, joined
back to `t1` through a current-user function. See "Query structure" above. Not expressible as a
prefilter column because the comparison value — whoever is logged in — isn't a static Filter value.

**4 · Relative date window** (query-based) — filtering to "this week," "overdue," "last 30 days" needs
a comparison against `getdate()` or a calendar/helper table, again not a static value. Keep the
comparison itself simple even when the surrounding subquery isn't — a plain `t1.due_date < getdate()`
is often enough without a helper table.

**5 · Look-up-only scoping** (either type) — set `look_up_prefilter_state` to On/On locked while
leaving Main/Detail at Off, to narrow a reference field's popup without touching the table's own
screen. Prefer On locked over On hidden here if the user should still see that the popup is filtered.

**6 · Data-quality flag** (query-based) — a correlated count against a table's own `_translated`-style
child rows, checked against the number of active application languages. See "Query structure" above;
generalizes to any "should have exactly N related rows" completeness check (required documents,
mandatory approval steps).
