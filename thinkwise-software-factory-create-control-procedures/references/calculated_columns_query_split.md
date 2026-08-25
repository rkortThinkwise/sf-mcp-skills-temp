# Calculated columns — and a wider 2026.2 `_query`-split pattern — are not physical

A column can be a **calculated/expression field** (`col.calculated_field_type` non-zero) rather than a
real stored column — and this is invisible from a normal read: it returns a value on every query
exactly like a physical column, with no visible difference in the entity's shape.

**Before including any column in a hand-written DML statement's column list, check
`col.calculated_field_type` for it** (`$select=calculated_field_type` against `col`): `0` means real
and writable; any other value means calculated, and the write must instead target whatever real table
actually backs it — check `calculated_field_query` on the same column to find that table and its key
(often a variant of the same key plus one extra dimension, e.g. `rdbms_type`).

**This same shape recurs much more broadly since 2026.2, on entities that have nothing to do with
`col.calculated_field_type`.** Multiple parent entities had whatever content could legitimately vary
per platform (a domain's data type, a query-based prefilter's SQL) split off into a companion
`<entity>_query` table keyed by the parent's key plus `rdbms_type`, while the parent entity's own
metadata still exposes the moved fields as ordinary-looking scalar properties. Reading them back
succeeds and shows real-looking values — that's exactly what makes this hard to see coming; nothing
about a normal read distinguishes a moved field from a real one.

Verified live, twice, on two different tables in the same session — each cost a failed generation
attempt before the real target was found:
- **`dom`** — `dttp_id`/`length`/`prec`/`dttp` (and the pre-2026.2 `prog_lang_id`, which no longer
  exists as a column at all) moved to **`dom_query`** (key: `model_id, branch_id, dom_id, rdbms_type`).
  `dom` itself keeps only `model_id, branch_id, dom_id` as its key. A dynamic-model procedure creating
  a brand-new domain now needs two inserts: one `dom` row (identity only), then one `dom_query` row
  per enabled `rdbms_type`.
- **`tab_prefilter`** — the `query` field (used only by a query-based prefilter, `prefilter_type = 0`)
  moved to **`tab_prefilter_query`** (key: adds `rdbms_type` to the parent's key). A column-based
  prefilter (`prefilter_type = 1`) never populates a query at all — the field should be dropped from
  the `#tab_prefilter` insert entirely, not set to `null`, since `null` is exactly what an old,
  now-removed column would have accepted.

**General detection technique, not specific to any one table**: before hand-writing DML against a
column of any entity, check that entity's metadata definition rather than trusting a normal read. If
a field's `kind` is `Scalar` (not `Key`) and the entity also exposes a
`detail_ref_<entity>_<entity>_query`-style navigation property, the field is very likely a computed
pass-through and the real writable data lives in that child entity — whose own primary key will
include `rdbms_type` where the parent's does not. The family observed in one model's metadata
includes (non-exhaustive, and likely to grow): `col_query`, `dom_query`, `tab_prefilter_query`,
`tab_check_constraint_query`, `indx_query`, `process_variable_query`, `report_parmtr_query`,
`task_parmtr_query`, `subroutine_parmtr_query`, `data_set_query`, `tab_query`, `unit_test_query`,
`tab_data_col_query`, `dom_input_constraint_query`. Don't assume the pre-2026.2 flat column name
still exists, and don't assume the metadata API's current property name for a field is the same as
the real column to write into — check the entity definition's `kind` and navigation properties first.
