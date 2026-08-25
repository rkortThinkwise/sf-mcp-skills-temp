# Subject presentation design — sort, search, filter, lookups, and views

This file is about *design*, not field mechanics: which pattern to pick for a subject's default sort,
which columns belong in search vs. filter, when a screen should be backed by a view instead of a
table, and how to design a lookup screen. For the actual `col`/`tab` fields (`default_sort`/`sort_no`,
`visible_for_search`/`search_condition`/`include_in_global_filter`,
`visible_for_filter`/`filter_condition`, `grp_until`, aggregation), see `SKILL.md`'s "Sort, search, and
filter" and "Grid row grouping and aggregation" sections — this file assumes those mechanics and adds
the reasoning behind the choice. For prefilters and variants as alternatives to a filter, see
`thinkwise_software_factory_prefilters` and `thinkwise_software_factory_variants`. For Grid/Form
column order, header groups, locking, and sections, see `thinkwise_software_factory_subject_components`.

## Start with the subject's job

Before tuning any column setting, state what this particular subject/variant is *for* — the same
underlying table commonly needs different sort/search/filter/columns for a different job, and forcing
one "complete" screen to serve every job at once is the most common source of an overloaded subject:

- Find and open one business object.
- Process a work queue.
- Compare several records.
- Enter or maintain a record.
- Review history or exceptions.
- Analyze totals and trends.
- Select a value in a lookup.

If a table needs to serve more than one of these jobs for different users, that's the signal to reach
for a variant (`thinkwise_software_factory_variants`) rather than compromise one screen to cover both.

## Default sort

### Pattern by subject type

| Subject type | Typical default sort |
|---|---|
| Master data | Recognizable name/code ascending, then primary key |
| Work queue | Priority/status, due or planned date ascending, stable key |
| Recent activity/history | Event timestamp descending, then unique key descending |
| Documents/orders | Business number descending for "latest"; customer/date/number for browsing |
| Sequence lines | Parent key, sequence/order number ascending |
| Calendar/planning | Start date/time ascending, resource/order as tie-breaker |
| Lookup | Display column ascending, then primary key |
| Exceptions | Severity or business priority, age/due date, stable key |

### Always make the order deterministic

If the leading sort columns aren't unique on their own, append a stable tie-breaker — normally the
primary key. Without one: paging can show duplicates or skip rows as data changes, a refresh can
reorder equal records, "next"/"previous" navigation feels random, and adding a row near the current one
becomes unpredictable. This is not optional polish — it's the difference between a sort that's
reproducible and one that silently isn't.

### Business meaning before technical convenience

Users rarely want a list ordered by an internal surrogate key or identity value. Put the recognizable
business name/code first unless recency is the screen's explicit purpose. The Software Factory's own
**Set up initial sort order** task defaults to primary keys and the lookup display column where
available — a sound baseline, but every high-value subject still deserves a deliberate business
review, not just the generated default.

### Ascending vs. descending

Ascending for names, codes, sequences, planned execution order, and upcoming dates. Descending for
logs, mutations, newly created documents, and latest status events. Be explicit about null placement —
should a missing value surface as an exception at the top, or sink to the bottom as incomplete data?

### Avoid poor sort columns

Disable or avoid a default sort on: binary/image/file/HTML/RTF or large multiline content, long free
text, expensive calculated expressions, deep-joined labels without a supporting index, values with no
useful human ordering, and sensitive values that shouldn't be exposed for comparison. Every column
technically allows sorting, but that permission isn't a best-practice endorsement — review large
subjects and expensive views specifically.

### Index alignment

For large datasets, make sure the common filter-plus-sort path is actually supported by a suitable
index: tenant/company + status + due date; customer + order date + order number; parent ID + sequence
number; resource + planned start. Don't create an index for every possible user sort — optimize the
default and frequent paths, then measure actual queries.

## Search: which fields belong in the search box?

The global search box searches several configured columns at once; a row matches when at least one
included field contains the entered value. It's available independently of the Filter permission.

### Strong candidates

Human-recognizable name/description; business number or code; customer/supplier/employee/project/
product/material/order/invoice/asset reference; barcode, serial number, license plate, external
reference, or tracking number; email/phone where that's a normal lookup route; a short alias or legacy
code users actually know; a concise combined display expression when it avoids searching several
redundant fragments.

### Usually better as filters than search

Status and category; active/inactive flags; boolean indicators; dates and date ranges; amounts,
quantities, percentages, rates; user/owner/resource selection; domain elements with a small known set.
These benefit from typed controls and correct operators — a user searching "open" shouldn't accidentally
match a description that happens to contain that word.

### Usually exclude entirely

Internal surrogate keys/GUIDs/hashes users never see; audit timestamps and technical users (unless
audit investigation *is* the subject's purpose); binary/file/image columns; large notes/HTML/RTF/
document bodies; encrypted/secret/sensitive data; expensive calculated values; repeated denormalized
fields that add no new search route.

### Design rules

Include the smallest set that covers how users actually identify records — don't add a column merely
because it's convenient for developers. Test with real search terms, including partial codes and
punctuation, not just clean example data. Order the included columns by usefulness — Universal UI
shows that order in the search tooltip, so put the most useful fields first and keep the translation
clear enough to explain the scope. Consider a dedicated indexed/normalized search column only once
measured performance or matching rules justify the extra complexity. For very large datasets, prefer
starts-with identifiers or structured filters over broad contains-matching across many expressions.

## Filtering

Filtering is structured narrowing over values users reason about as conditions, not text fragments.

### Always-visible candidates — keep this set short and task-oriented

Status/workflow state; owner, assignee, team, customer, supplier, or resource; the relevant date/date
range (due, planned, delivery, invoice, created); main category/type; active/inactive when users
genuinely switch between them; location, warehouse, project, or business unit.

### Extended candidates

Secondary classifications; audit user/date fields for power users; less common references; numeric
ranges; secondary flags/lifecycle dates; technical integration state exposed to support users. Audit
fields are a frequent modeling pattern but usually belong in Extended, not Always, for daily
operational filters.

### Never-filter candidates

Files/images/binary/HTML/RTF/unsupported control types; long narrative text (search is more suitable);
columns hidden for security; values needing an unacceptably expensive calculation/join; duplicate
representations of the same concept.

### Default filter condition

Text identifiers: starts-with or contains, based on size/indexing. Exact codes/domain elements: equals
or in. Dates/numbers: equals only when exact matching is normal; otherwise ranges/comparisons. Booleans:
equals. Nullable status: deliberately support empty/not-empty where that state is meaningful.

### The filter form is stricter than the filter popup

The filter form renders whatever conditions were modeled — users can't change its operators at
runtime, unlike the flexible filter popup. Its column selection and conditions therefore need a
stricter design review; a wrong default operator there can't be worked around by the end user the way
a popup mismatch can.

### Related-subject (deep-join) filtering

Filtering through lookup/detail references ("orders for customers in region X") is powerful but every
extra join level adds cognitive and query cost. Offer related fields that answer common business
questions, not every reachable path, and inspect the actual generated query on a large dataset before
shipping a deep filter.

## Tables vs. views, for presentation reasons

Use a **table** subject when users maintain the business entity directly. Reach for a **view** subject
instead when the screen needs: denormalized lookup labels and derived status, a work queue joining
several entities, aggregated/reporting data, a stable read model optimized for search/filter/sort, or a
role-/process-specific projection that shouldn't leak into the base table's own screen. See
`thinkwise_software_factory_create_view` for how to actually build the view once this is the answer.

A view built for presentation should still expose a stable identity and a deterministic sort — the same
rules above apply to it, not a relaxed version. Avoid views with ambiguous update semantics; if the view
must be editable, define its handlers/process logic deliberately and test concurrency (see
`thinkwise_software_factory_create_control_procedures`'s Handler guidance). Don't duplicate every base
column into the view "just in case" — project only what identification, decisions, filters, sorting,
and actions actually need.

## Lookup subjects

A lookup screen has one narrow job: help the user identify and choose a single record.

- Set a meaningful `tab.look_up_display_col_id` (see `SKILL.md`'s "Look-up display column") and sort by
  that display value, with the primary key as tie-breaker.
- Include the code, name, and any common alternate identifier in search — a lookup is exactly the
  screen where users type a fragment they already know.
- Add status/category/location filters only when they genuinely help narrow the choice, not by default.
- Apply a hidden or locked prefilter to remove records that are structurally invalid choices for this
  context — see `thinkwise_software_factory_prefilters`'s "look-up-only scoping" pattern, which does
  exactly this without touching the table's own main screen.
- Use a compact variant rather than the full maintenance screen for the popup itself.
- Prefer Suggestions (starts-with) for large datasets; a combo control suits small, stable datasets.
  Contains-style suggestions can improve discovery but cost more — reserve them for cases where
  starts-with genuinely isn't enough.

## Testing

- **Sorting**: equal leading values stay in stable key order; nulls appear where expected; paging and
  refresh don't duplicate/skip records; new rows appear predictably; large data uses an acceptable
  query plan.
- **Search**: real names, partial codes, serials, email, and references find the expected records;
  technical/irrelevant values don't unexpectedly match; special characters, Unicode, casing, and
  leading zeros behave correctly; tooltip order/translations explain the scope; large datasets respond
  within target time.
- **Filter**: Always/Extended fields actually match real usage frequency; default operators are useful
  per data type; header/quick/popup/filter-form/prefilter/related filters combine correctly; clearing
  filters removes hidden-column filters too; deep joins respect authorization and performance limits.

## Anti-patterns

- **Primary-key-only default sort** — stable but meaningless to users.
- **Non-deterministic date sort** — equal timestamps jump between pages with no tie-breaker.
- **Search everything** — slow queries and surprising matches from over-broad search scope.
- **Search as a status filter** — ambiguous text matching standing in for a typed control.
- **Every filter Always** — the basic filter popup grows into a second full form.
- **Audit-field dominance** — daily users see insert/update metadata before anything they actually need.
- **Deep join without measurement** — related filters that are fine in development and slow in
  production.

## Final review checklist

- Does the default sort express business priority and remain deterministic?
- Are frequent filter+sort paths indexed appropriately?
- Can users find records by every identifier they actually know?
- Is search limited to useful, safe, performant fields, in a sensible tooltip order?
- Are status, dates, categories, and ranges handled as filters rather than loose search text?
- Is the Always filter set short, with secondary fields Extended?
- Have deep/related-subject filters been checked against authorization and query cost?
- Is this subject presented as a table or a view for the right reason — not just habit?
- Does a lookup variant of this subject actually narrow to valid choices, with a meaningful display
  column and sort?
