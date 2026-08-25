# Cube-level `cube_view_field_conditional_layout` — field reference (verified)

Keyed by `(model_id, branch_id, cube_id, cube_view_id, cube_field_id, conditional_layout_id)`. This
family is structurally different from the table/task families: **the condition lives inline on the same
row** — there is no separate `cube_view_field_conditional_layout_condition` child entity (confirmed
absent) — so one cube conditional layout is exactly one condition.

| Field | Notes |
|---|---|
| `condition_cube_field_id` | Which field the condition evaluates — independent of the row's own `cube_field_id` (the field being styled), same "target ≠ evaluated" pattern as the table family |
| `numeric_condition` | A **10-value** subset of the general condition enum — verified: `equal_to`=0, `not_equal_to`=1, `greater_than`=2, `smaller_than`=3, `greater_than_or_equal_to`=4, `smaller_than_or_equal_to`=5, `between`=6, `not_between`=13, `is_empty`=10, `is_not_empty`=11. No contains/starts-with/in family — cube values are numeric measures |
| `value` / `until_value` | Constant comparison value(s) — no `value_col_id`/column-comparison option was found on this entity, unlike the table/task condition children |
| `apply_to_cell` / `apply_to_total_cell` / `apply_to_custom_total_cell` / `apply_to_grand_total_cell` | Four independent surfaces — a layout can target ordinary cells, subtotal cells, custom-total cells, and/or the grand total independently |
| `gradient_direction_4` | `none`=0 · `vertical`=1 · `horizontal`=2 · `backward_diagonal`=3 · `forward_diagonal`=4 — cube layouts support a genuine two-colour **gradient**, which the table/task families don't have |
| `background_color_1` / `background_color_2` | Gradient endpoints (legacy, non-themed `Edm.Int32`) |
| `background_color_light` / `background_color_dark` | Universal UI themed background (flat, non-gradient) |
| `font_color_light` / `font_color_dark`, `bold`/`italic`/`underline`/`strikethrough`, `font_size` | Same styling vocabulary as the other families |
| `image_id` / `image` | A cube-layout-only capability — attach an icon/image, not just colour/text styling. `image_id` is a foreign key into the same shared `icon` repository used everywhere else (confirmed live via its `icon.icon_look_up` navigation) despite the "image" naming — pick a suitable icon per `thinkwise_software_factory_icons`'s status vocabulary (unique silhouette per condition, never color alone) rather than leaving it unset, and ask the user rather than guess if nothing in the repository fits. |

Prefer threshold formatting (negative margin, variance against budget, values outside tolerance, service
level below target) over assigning a different colour per category — the gradient support exists
specifically for continuous-value emphasis (heatmap-style), not for distinguishing discrete categories.

**Known gap, verified**: `cube_field_conditional_layout`/`cube_field_total` — an older, `cube_field`-
scoped pair distinct from `cube_view_field_conditional_layout`/`cube_view_field_total` — exist as real
translatable object types in the model but were not reachable as entity sets through this connector.
Treat `cube_view_field_*` as the current, Universal-UI-facing mechanism; if a model surfaces the legacy
`cube_field_*` pair, that's the older/Windows-GUI-only cube object, not a distinct feature to build new
work against.

Bound tasks: `task_show_history`, `task_unlink_generated_object` (no copy/rename/delete — each row is a
complete, self-contained layout+condition, so it's added/removed as a whole rather than composed from
parts the way the table/task families are).
