# Converting a user's ad hoc query into a view

Use when a user hands over a working SQL query (from SSMS, a report tool, a BI dashboard) and asks
for it to become a proper view. Follow "Confirm scope before modeling", "Setting up a view — the
basics", and "Writing and assigning code to a view" above step-for-step; the differences are:

- **Start from the query's intent and grain, not its text** — what business question does it answer,
  what does one output row represent. Settle this before modeling anything; it drives the primary-key
  choice below.
- **Confirm every source table/column against the live model** (`get_entity_definition`/
  `get_domain_definition`) before reusing it, rather than trusting the raw SQL's names — the query may
  reference an old alias, a renamed column, or a table since split. Pin down the exact
  `tab_id`/`col_id`/`dom_id` for each before proceeding.
- **Domain reuse is the step most likely to get skipped under time pressure** when starting from
  someone else's SQL — don't skip it; this is exactly the situation that produces one-off domains if
  nobody checks (see "Reusing columns and domains" above).
- **Primary key**: derive it from the query's actual grain, not habit. A `GROUP BY` almost always
  gives it directly; a straight join with no aggregation usually keeps the driving table's PK (plus
  any 1:many join key that fans out the grain). Verify uniqueness against real data either way.
- **Name the view for the business concept it represents, never the query's origin** — not
  `user_report_query` or `temp_view_1`, but e.g. `open_order_backlog`.
- **When translating the SQL into the control procedure template** (step 5 of "Writing and assigning
  code"): rewrite every table/column reference to the confirmed `tab_id`/`col_id`s above rather than
  pasting the user's raw SQL verbatim; match the *model's* dialect and formatting conventions, not
  whatever tool the query came from (SSMS implies SQL Server, which may not be this model's platform —
  translate into every `rdbms_type` the model actually uses, per step 1); and preserve the query's
  filtering/join logic faithfully — flag any semantic change (e.g. an inner join the user wrote as a
  left join) back to the user rather than silently fixing it.
- **Verify by diffing, not just a clean generate**: compare the generated view's SQL logic against the
  user's original query — same joins, same filters, same grain — before calling the conversion done.
- **Sequence data before UI.** Only once the view itself is verified correct, wire up any GUI the user
  also asked for (a screen, a grid, a report data source) — keeping that separate means a UI issue is
  never confused with a data bug.
- **Poor-fit flagging applies here too** — see "Confirm scope before modeling" above; a query that's
  a poor fit for a plain view doesn't stop being one just because it arrived as ready-made SQL.
