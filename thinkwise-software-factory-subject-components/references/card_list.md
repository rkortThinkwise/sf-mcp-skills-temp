# Card List

## What it is, and when to use it

A Card List shows rows as vertically stacked cards with a deliberately small column set — a
scan-and-select component best suited to mobile/field use and any screen where selecting a record
matters more than comparing columns across many rows. In a detail tab it automatically limits itself
to the parent's related rows.

Use it for mobile/field-service queues, contacts/customers/products with a strong visual identity,
search results where selection dominates, or a master list paired with a Form for the real detail.
Prefer a **Grid** when users need to compare many columns or bulk-edit; prefer a **Form** for one
record with many fields; prefer a **Treeview** for hierarchical/grouped navigation.

## Field reference — verified

`tab` / `tab_variant_overview`:

| Field | Type | Purpose |
|---|---|---|
| `card_list_title` | enum: `none`=0, `first_visible_field`=1, `look_up_display`=2 | Source of the card's heading. |
| `card_list_image` | enum: `none`=0, `first_visible_image_field`=1 | Source of the card's image. |

`col` / `tab_variant_card_list(_overview)`, keyed by `(…, tab_id[, tab_variant_id], col_id)`:

| Field | Type | Purpose |
|---|---|---|
| `card_list_order_no` | int | Display order on the card. |
| `card_list_type_of_col` | enum: `editable`=0, `read_only`=1, `hidden`=3 (**no value 2** — same gap on grid/form type enums) | Visible/editable state, capped by the underlying column definition. |
| `card_list_label` | flag | Whether the field's label renders next to its value. |
| `card_list_field_height_in_positions` | int | Extra vertical space, for a short description/address that needs wrapping. |

`tab_variant_card_list_overview` adds read-only `type_of_col`/`calculated_field_type` and bound tasks
`task_move_card_list_col_up`/`_down`, `task_move_tab_variant_card_list_overview_card_list_order_no`,
`task_renumber_tab_variant_card_list`.

**Verified live example** (`INSIGHTS.employee`): `card_list_title = first_visible_field`,
`card_list_image = first_visible_image_field`; columns ordered `employee_id`(10) → `name`(20) →
`employee_function_id`(30, read-only) → `hourly_rate`(40, hidden, labelled) → `photo`(50, image
field) → `first_name`/`last_name`/`prefix`(60–80, hidden) → … . `hourly_rate` is the one field with
`card_list_label = true` in this table — exactly the "value is ambiguous without a label" case
(a bare number could be anything; the rest read fine unlabeled because their format/position already
say what they are).

## Step by step

1. Decide the title source: `look_up_display` is the strongest default for named/numbered business
   entities (it's already the identity shown everywhere else); `first_visible_field` only when the
   card genuinely needs a different heading *and* the first visible field is reliably populated;
   `none` only when another element unambiguously carries identity. If this isn't obvious from the
   request, ask the user rather than defaulting to `look_up_display`.
2. Decide the image source: `first_visible_image_field`, only when it materially helps recognition —
   never decorative.
3. For each column that should appear on the card: set `card_list_order_no`, `card_list_type_of_col`,
   and `card_list_label` (label only where the bare value would be ambiguous). Add
   `card_list_field_height_in_positions` only for the one field that genuinely needs wrapping.
4. Keep the visible set small — a practical guideline (not a platform limit) is roughly four to seven
   fields: identity, status/exception, the primary operational fact, next time/action, location/owner.
5. Configure a meaningful default sort on the underlying subject (`col.default_sort`/`sort_no`) — a
   good card layout can't fix a poorly ordered queue.
6. Place a Card List leaf panel on the subject's screen type.
7. For a variant that needs a different card layout, run `task_setup_tab_variant_card_list_overview`
   first (per `thinkwise_software_factory_variants`), then edit `tab_variant_card_list`.

## Best practices and pitfalls

- **Every field answers one of**: what is this / what state is it in / what matters now / where or
  with whom. Drop anything that doesn't.
- **Hiding a field on the card is presentation, not security** — enforce real restrictions at
  role/subject/column level.
- **Reordering visible fields can silently change `first_visible_field`'s title or the selected
  image** — both are "first visible", so a later column-visibility change is a regression risk; include
  identity/image checks in regression testing after any card-list reorder.
- **Gotcha — no dedicated conditional-layout flag for Card List either** (same finding as Tree above).
  Verify empirically whether/how conditional formatting shows up on a card before depending on it.
- **Variant drift**: after any base-column change, re-check every effective variant's title/image
  selection, field set, order, and labels — these can drift independently per variant.
- **Common failure**: a card with too many fields and long text becomes a form in disguise and loses
  the scan-and-select benefit entirely — route long content to the paired Form instead.
