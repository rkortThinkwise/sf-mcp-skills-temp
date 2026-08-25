# Form

## What it is, and when to use it

The Form displays and edits one row in full — labels, controls, validation, grouping — normally the
row currently selected in a Grid/Card List/Treeview. Use it for master data, guided data entry with
defaults/lookups/validation, or any record with a mixture of control types that needs more space than a
Grid row or a card can offer. Prefer **Grid**/**FormList** for rapid multi-row editing of a few
columns; prefer a **Task** for a short, parameter-driven operation rather than a persistent record edit.

## Field reference — verified

`tab` / `tab_variant_overview` (subject-level geometry and behaviour):

| Field | Type | Purpose |
|---|---|---|
| `no_of_cols_in_form` | int | Max form columns Universal UI may render; `0` = as many as fit. |
| `form_col_min_width_factor` | decimal | Relative width given to form columns. |
| `label_width` / `field_width` / `field_height` | int (px) | Baseline geometry. |
| `height_between_fields` / `width_between_fields` | int (px) | Spacing/rhythm between fields. |
| `form_default_editable` | flag | Auto-edit — Form opens already in edit mode. |
| `form_auto_save` | flag | Auto-save on field change (distinct from auto-edit). |
| `hide_buttons` | flag | Hides the Form's own nav/update buttons (actions remain via the action bar). |

`col` / `tab_variant_form(_overview)` (per-field placement and behaviour):

| Field | Type | Purpose |
|---|---|---|
| `form_order_no` | int | Field order. |
| `form_type_of_col` | enum: `editable`=0, `read_only`=1, `hidden`=3 | Never more permissive than the underlying column. |
| `field_height_in_positions` | int | Extra height (multiline notes, addresses, images). |
| `field_no_of_positions_further` | int | Relative spanning/positioning. |
| `field_in_next_col` | flag | Moves the field into another form column, space permitting. |
| `form_field_in_next_grp` / `form_next_grp_label` / `form_next_grp_icon_id` | flag / string / icon | Starts a new **group** (a labelled sub-heading within the same section). |
| `field_on_next_tab` | flag | Starts a new **section** — Universal UI renders these vertically stacked, not as literal tabs, despite the `next_tab` naming. |
| `next_tab_label` / `next_tab_icon_id` | string / icon | Section label/icon (carried by the section's first field). |
| `next_tab_default_expanded` | flag | Off = section starts collapsed, showing no fields until expanded. |
| `form_col_max_space` (read-only) | decimal (%) | Universal UI's calculated per-field size, derived from field/label width and the model's floating-label strategy — informational, not directly settable. |

**Unverified aside**: `col` also carries a `section` field (`Edm.Int16`, enum, values like
`start_tab_same_grp_start_line` encoding tab/group/line position as one packed number). It correlates
exactly with `field_on_next_tab` / `form_field_in_next_grp` / `field_in_next_col`, and is very likely a
computed/derived summary of those three booleans rather than something to set directly — not
independently confirmed, so verify before writing to it directly instead of through the three booleans.

## Groups vs. sections

| | Group | Section |
|---|---|---|
| Fields | `form_field_in_next_grp`, `form_next_grp_label`, `form_next_grp_icon_id` | `field_on_next_tab`, `next_tab_label`, `next_tab_icon_id`, `next_tab_default_expanded` |
| Renders as | A labelled sub-heading within the current section | A new vertically-stacked block (Universal) / literal tab (legacy Windows GUI) |
| Collapsible | No | Yes, via `next_tab_default_expanded` |
| Use for | Related fields within one concept (address block inside "Contact details") | Genuinely independent concepts (core vs. optional, current vs. historical) |

Use **Field in next column** (`field_in_next_col`) when the intent is purely positional, not semantic —
don't fake a column break with an empty group.

**Verified live gotcha**: `form_next_grp_label` and `next_tab_label` silently convert spaces (and `&`)
to underscores server-side when patched — e.g. "Account details" persists as `Account_details`, and
"Type & default" as `Type___default` — regardless of `value_kind` (`data` doesn't avoid it). This is
not a display artifact; `execute_odata_query` confirms the underscored value is what's actually stored
in `col`. Setting either label field also auto-creates a placeholder row in `transl_object_transl`
(key: `model_id`, `branch_id`, `type_of_object=33` — confirmed identical for both group and section/
next_tab labels, no separate type for sections — `transl_object_id`=the underscored label,
`appl_lang_id='en-US'`) with `transl` set to a bracketed placeholder like `[Account_details]` and
`approval_status=0`. The bracket placeholder, not the underscored code, is what actually renders in
the UI until fixed — after every new group/section label, `stage_resource(action="edit",
entity_set="transl_object_transl", key={...})` → `patch_resource` to set `transl` to the real
human-readable text (spaces/`&` intact) and `approval_status=1` → `commit_resource`. `transl_object_transl`
rows are scoped by `model_id`+`branch_id`+`transl_object_id`, not per table/column — if the same label
text is reused across tables (e.g. "Name" on two different forms), fix its translation once and check
via `execute_odata_query` before repeating the work for a second table using the same label.

## Variant overrides and a useful shortcut

`tab_variant_form` / `tab_variant_form_overview` mirror the column-level fields above, keyed by
`(…, tab_id, tab_variant_id, col_id)`. **`tab_variant_form_overview` exposes `task_copy_grid_to_form`**
(and the Grid side exposes `task_copy_form_to_grid`) — each takes `order_no`/`visibility` flags and
copies that aspect straight from the other component, which is much faster than re-typing a parallel
field order by hand when a Grid and Form should track each other closely.

## Step by step

1. Set subject geometry conservatively: `no_of_cols_in_form` (0 or a deliberate small number),
   `label_width`/`field_width`, `field_height`, spacing.
2. For each field: set `form_order_no` by workflow (identity → status/ownership → primary
   input/decision → dates/references → additional detail → audit, last and usually read-only), and
   `form_type_of_col` explicitly — never leave a field's editability to accident.
3. Group tightly related fields (`form_field_in_next_grp` + label/icon); start a new section
   (`field_on_next_tab` + label/icon) only at genuine concept boundaries; collapse secondary/advanced
   sections with `next_tab_default_expanded = false` — never the primary task or a section with active
   validation errors. When a group/section gets an icon, set `form_next_grp_icon_id`/`next_tab_icon_id`
   to a suitable one per `thinkwise_software_factory_icons`'s form-navigation guidance (simple,
   recognizable at ~16px) rather than an arbitrary pick — but per that skill's own guidance, an icon
   isn't mandatory on every group; add one only where it earns its place (collapsed sections, repeated
   patterns across forms). **Cover the first visible field too** — the leading identity/PK column(s)
   that come before the first semantic group are still part of the Form and need a group of their own
   (e.g. an "Identity" group covering the record's own id and any leading display-name field), not left
   stranded outside any group just because they happen to come first. If a leading column is hidden
   (`form_type_of_col = hidden`), the group must start at the first field that's actually *visible*, not
   at the hidden one — see the "Hidden group starter" pitfall below.
4. Use `field_in_next_col` / `field_no_of_positions_further` for deliberate column layout, not dummy
   spacer fields or padded labels (both break under translation and responsive reflow).
5. Decide `form_default_editable`/`form_auto_save` together as one interaction model — auto-edit
   everywhere invites accidental changes on review-heavy screens. If the interaction model isn't
   obvious from the request, ask the user rather than defaulting both to off (view-first, manual save).
6. Place a Form leaf panel on the subject's screen type.
7. For a variant, run `task_setup_tab_variant_form_overview`, then either hand-edit
   `tab_variant_form`/`_overview` or seed it fast with `task_copy_grid_to_form`.

## Best practices and pitfalls

- **Database-order forms are a smell** — order by the user's task, not table-creation history.
- **Everything editable / everything visible** are the two most common failure patterns — set every
  field's type deliberately.
- **A mandatory-but-dynamically-hidden field** breaks save with no visible cause — when layout logic
  drives visibility, test every combined mandatory×visible state.
- **Hidden group starter**: if a hidden field happens to be the one carrying `form_next_grp_label`, the
  group's own label/icon can go missing — check the Software Factory's "Possibly visible" prefilter
  when a group heading looks wrong.
- **Hidden/read-only is presentation, not security** — enforce real restrictions independently.
- **Auto-edit is not auto-save** — review both together, and test accidental-change/concurrency
  behavior before enabling either broadly.
- **Good candidates to collapse by default** (`next_tab_default_expanded = false`): audit/trace fields
  (created/modified by-and-on), integration request/response detail, advanced or rarely-changed
  settings, long notes/history/diagnostics not needed for the primary task, completed-process detail
  relevant only during investigation, optional attachment/metadata areas on a very large form.
- **Keep expanded by default**: identity and current status, required fields, fields users normally
  edit in this task, validation errors (or whatever the user needs to understand them), any
  safety/compliance/approval consequence, and the record's key outcome. A validation error landing
  inside a collapsed section is a real risk — test that the UI expands or otherwise surfaces it rather
  than hiding the very thing the user needs to fix.
- **Grouping isn't a one-time setup task** — revisit it whenever a Form is placed on a table for the
  first time, and every time a new column is added to a table whose Form already has groups/sections.
  Match the new field to whichever existing group covers the same concept and set its `form_order_no`
  adjacent to that group's other members, in the same change that adds the column — don't default to
  appending it ungrouped at the end. If no existing group is an obvious semantic fit, ask the user
  which group it belongs in (or whether it needs a new one) — see
  `thinkwise_software_factory_mcp_base`'s "Ask, don't default" rule — rather than guessing or creating
  a new group unilaterally. This includes the **first visible column** of the Form — it's easy to
  treat a leading identity/PK field as exempt from grouping because it "comes before" the first
  semantic group, but a Form where every field is grouped except the very first one is still an
  incomplete grouping pass, not a finished one.
