---
name: thinkwise-software-factory-maps-component
description: Reference guide for setting up and maintaining a maps component in a Thinkwise Software Factory model — the map/map_base_layer/map_overlay/map_data_mapping entities, the CoordSets JSON coordinate contract, and per-domain-element styling. Use before calling get_entity_definition/execute_task/execute_odata_query against map/map_base_layer/map_overlay/map_data_mapping/tab_variant_map*, or before writing a control procedure that builds or parses CoordSets JSON, via an MCP connector with Software Factory access (e.g. sf_mcp, indicium).
---

# Setting Up and Maintaining a Maps Component in the Thinkwise Software Factory

Reference for the maps component's full lifecycle: source column → `map` → `map_base_layer`/
`map_overlay` → `map_data_mapping` → (optionally) drawable-shape task wiring → (optionally) an
external-API pipeline that populates the map's coordinates. Every entity/field name below was
confirmed live against a real model (`sf/manage_maps` domain, 32 entity sets) and against a working
reference application (`MAPS_AND_ROUTES`) that implements address geocoding and route calculation —
not guessed from documentation.

Apply this whenever an MCP connector with Software Factory access (`sf_mcp`, `indicium`) is used to
create, inspect, or troubleshoot a maps component — follow the connector's standard discovery→act
flow; never guess entity/task/property names.
`map`/`map_base_layer`/`map_overlay`/`map_data_mapping` typically live in a `manage_maps`-style
domain; the map's source `tab`/`col`/`dom`/`ref` live in a `manage_datamodel`-style domain;
`control_proc`/`control_proc_template` live in a `manage_control_procedures`-style domain;
`web_connection`/`web_connection_endpoint` live in a `manage_webconnections`-style domain;
`process_flow`/`process_action` live in a `manage_process_flows`-style domain — try these directly
first, and only escalate to `search_capabilities`/`get_available_domains` on an
`entity_set_not_found`/`domain_not_found`-style rejection rather than re-discovering a domain that
already resolved earlier this session.

For general data-modeling rules (naming, domain reuse, reference direction) see
`thinkwise_datamodeling_guidelines`. For control-procedure mechanics (code groups, static vs. SQL
assignment, `branch_rdbms_type`, and — critically — the two-step "generate code group" then
"generate object code" sequence) see `thinkwise_software_factory_create_control_procedures`; every
control procedure referenced below is created and generated exactly that way. This skill only covers
what's specific to maps.

## What a maps component is

A maps component renders one table's rows as geographic content — pins, lines, circles, rectangles or
polygons — on a tile-based map (Leaflet), modeled once per table (`tab_id`) and reused by every screen
that shows it. It is built from four cooperating entities, all keyed by `(model_id, branch_id,
tab_id, …)`:

| Entity | Cardinality | Purpose |
|---|---|---|
| `map` | one per table | Center/zoom, and which columns feed coordinates, styling, popup, label, and drawn-shape capture |
| `map_base_layer` | many per table | The tile provider(s) underneath everything |
| `map_overlay` | many per table | Optional extra tile layer(s) on top, independently togglable |
| `map_data_mapping` | many per table, keyed by `(dom_id, map_data_mapping_id)` | How each row's geometry is styled, one row per domain element |

A map both **displays** data (rows → styled shapes, driven by `map_data_mapping`) and can **capture**
it (a user draws a shape → a task writes it back, driven by the `*_location_task_parmtr_id` fields on
`map` — see "Drawable shapes" below). These are two independent, optional wiring paths on the same
`map` record; a table can use either, both, or neither (e.g. a purely server-calculated map, like the
route example below, wires neither).

Every one of these four entities also has a `tab_variant_*` counterpart (`tab_variant_map`,
`tab_variant_map_base_layer`, `tab_variant_map_overlay`, `tab_variant_map_data_mapping`) that
overrides the table-level defaults for one specific tab variant — see "Tab-variant overrides" below.

## The CoordSets JSON contract

The map does not read raw latitude/longitude columns directly. `map.latitude_longitude_col_id` must
point at a column producing a small JSON envelope, referred to as **CoordSets**, structured as a list
of coordinate rings — one shape for a point, a line, or a multi-ring polygon:

```json
{ "CoordSets": [ [ { "Lon": "5.979388", "Lat": "52.208479" } ] ] }
```

- A **marker** or **circle** has exactly one ring with exactly one point.
- A **line** has one ring with multiple points, in path order.
- A **polygon**/**rectangle** has one or more rings (outer boundary, optional inner holes).
- A **circle** additionally carries a sibling `"Radius"` key (meters) alongside `"CoordSets"` — see
  "Drawable shapes" below for the full shape.

This column is almost always a **calculated field** (`col.calculated_field_type = expression`,
`col.calculated_field_query` holding the SQL) so it always reflects the row's live coordinates without
a stored duplicate. Verified example, straight from a production table (SQL Server dialect):

```sql
-- col: address.map_entity_coordinates, domain: varchar_max
concat('{ "CoordSets": [ [ { "Lon": "', t1.longitude, '", "Lat": "', t1.latitude, '" } ] ] }')
```

**Check `branch_rdbms_type` before writing this kind of SQL.** See the
`thinkwise_software_factory_create_control_procedures` skill for the full dialect-check walkthrough;
the essential rule is: never assume SQL Server syntax works on PostgreSQL, or vice versa — verify
first. Specifically for CoordSets: `concat(...)` works as shown on SQL Server; on PostgreSQL the
equivalent is typically `json_build_object('CoordSets', json_build_array(json_build_array(
json_build_object('Lon', t1.longitude, 'Lat', t1.latitude))))::text` or an equivalent
`jsonb_build_object`/`format()` construction.

A calculated field is not the only option: for a table whose geometry is written by an external
process (route calculation, geocoding, a drawn-shape task — see later sections), `map_entity_coordinates`
is instead an ordinary stored column that a control procedure `update`s directly with a hand-built
CoordSets string.

**Transposing `Lon`/`Lat` is a silent, hard-to-notice bug — sanity-check it against a real place.**
Both are ordinary numeric values sitting next to each other in the same `concat`/`json_build_object`
call, so swapping which source column feeds which JSON key produces a string that is still perfectly
valid CoordSets JSON and generates without any error — it just plots every row in the wrong spot (a
verified case: real-world coordinates with `Lon`/`Lat` swapped rendered a Netherlands address in the
Indian Ocean off the Horn of Africa). After writing this expression, check the map against at least
one row whose real-world location you actually know, not just that the column generated and the map
renders *some* pin.

## `map` — field reference

Keyed by `(model_id, branch_id, tab_id)` — one row per table that has a maps component.

| Column | Purpose |
|---|---|
| `initial_latitude` / `initial_longitude` / `initial_zoom_level` | Where the map centers when first opened, if not auto-fitting to data |
| `latitude_longitude_col_id` | The CoordSets-producing column (above) |
| `data_mapping_col_id` | The domain-based column selecting which `map_data_mapping` style applies to a row — see next section for the exact matching contract |
| `popup_col_id` | Column shown in the shape's popup on click — often an HTML calculated field |
| `use_custom_label_col_id` (flag) / `label_col_id` | Optional always-visible label per shape, instead of only-on-click popup content |
| `marker_location_task_parmtr_id`, `line_location_task_parmtr_id`, `polygon_location_task_parmtr_id`, `rectangle_location_task_parmtr_id`, `circle_location_task_parmtr_id` | **Geometric Type Creation** wiring — which task parameter receives the coordinates when a user draws that shape type on the map. Leave all five unset for a purely display/server-calculated map (verified: the reference model's two maps used none of them). See "Drawable shapes" below |

**Lookup restriction on the five `*_location_task_parmtr_id` fields**: their navigation properties
target `task_parmtr.only_alphanumeric` — only task parameters passing that filter are selectable.
Confirm a candidate parameter satisfies it (via `get_entity_definition`/`get_domain_definition`
before assigning; don't assume a parameter is eligible just because it exists) rather than assuming
any task parameter can be wired in.

## `map_base_layer` / `map_overlay` — field reference

Both are XYZ tile layers with the same shape; overlays add opacity and menu visibility since they're
meant to be independently toggled on top of a base layer. Keyed by `(model_id, branch_id, tab_id,
map_base_layer_id)` / `(…, map_overlay_id)`.

| Column | Purpose |
|---|---|
| `map_base_layer_uri` / `map_overlay_uri` | XYZ tile URL template, e.g. `https://…/{x}/{y}/{z}` or a provider-specific query string form |
| `order_no` | Stacking order among multiple layers |
| `min_zoom_level` / `max_zoom_level` | Zoom range **the tileset itself serves** — not a client-side UI limit. Setting this higher than the provider actually supports returns blank/upscaled tiles, it does not unlock more detail |
| `tile_size` | Pixel size of each tile, if the provider deviates from the 256px default |
| `attribution_html` | Attribution shown in the map's corner — mandatory for most public tile providers' terms of use |
| `show_map_base_layer` / `show_map_overlay` | Default visibility |
| `opacity` *(overlay only, `Edm.Byte` 0–255)* | Transparency so the base layer stays visible underneath |
| `show_in_menu` *(overlay only)* | Whether users can toggle this overlay themselves |

**If the user hasn't specified which tile provider(s) to use, ask** — per
`thinkwise_software_factory_mcp_base`'s "Ask, don't default" convention, don't pick one unilaterally
just because a URL template is easy to construct.

Verified example — three base layers on one table, sharing one URL template with only the `lyrs` query
parameter changed (Google classic tile endpoint: `m` = road, `p` = terrain, `y` = hybrid/satellite),
all zoom 0–18:

```
road:      https://mt0.google.com/vt/lyrs=m&hl=en&x={x}&y={y}&z={z}
terrain:   https://mt0.google.com/vt/lyrs=p&hl=en&x={x}&y={y}&z={z}
satellite: https://mt0.google.com/vt/lyrs=y&hl=en&x={x}&y={y}&z={z}
```

Overlays are optional — reach for one only when a layer needs to be independently toggled and
opacity-blended over whichever base layer is active (traffic, weather, custom raster/heat data).

**Bearer/OAuth2-token-authenticated tile sources are not supported directly.** The component is built
for a long-lived token embeddable in the URL (a query-string API key, as above). A short-lived bearer
token needing an OAuth2 refresh flow can't be handled by the browser/server tile request — front it
with your own proxy web service that injects and renews the header, and point `map_base_layer_uri` /
`map_overlay_uri` at the proxy instead of the token-protected source directly.

## `map_data_mapping` — field reference and the domain-element matching contract

Keyed by `(model_id, branch_id, tab_id, dom_id, map_data_mapping_id)` — **one row per element of the
domain referenced by `map.data_mapping_col_id`.** The matching contract, verified against a real
model: the column `map.data_mapping_col_id` points at must use domain `dom_id`; each
`map_data_mapping_id` must equal one of that domain's element IDs (`elemnt.elemnt_id`). A domain with
4 elements (verified: an `IMAGE_COMBO`-controlled `varchar` domain, `no_of_elemnt = 4`) needs exactly
4 `map_data_mapping` rows to give every element a style — an element with no matching row has no
defined rendering.

| Column | Purpose |
|---|---|
| `geometric_type` (`Edm.Byte`, enum) | `marker` = 0 · `line` = 1 · `polygon` = 2 · `rectangle` = 3 · `circle` = 4 |
| `border_color` / `fill_color` (`Edm.Int32`) | **Signed 32-bit ARGB** — see decoding formula below |
| `border_width` | Outline width |
| `border_opacity` / `fill_opacity` (`Edm.Byte`, 0–100) | Separate from the color's own alpha channel — both are applied |
| `allow_drag_drop` (flag) | Lets a user reposition this shape by dragging it on the map, writing the new position back through drag/drop logic — **not** the same mechanism as the generic `drag_drop`/`drag_drop_parmtr`/`drag_drop_matrix` entity family covered in `thinkwise_software_factory_subject_components`; this is a map-specific flag with no task/parameter wiring of its own |
| `show_map_data_mapping` (flag) | Default on/off state for this style's legend/layer entry — **must be turned on for the style to actually render on the map at all**; don't assume it defaults to visible, verify it explicitly on every row |

### A `marker` row ignores all its color/width/opacity fields — style comes from the element's icon instead

**Verified directly against the platform's own Layout and Default control procedures for
`map_data_mapping`** (read from the Software Factory's own meta-model, not inferred): the moment
`geometric_type` is set to `marker` (0), a Default-type control procedure clears `border_color` and
`fill_color` to `null` and resets `border_width`/`border_opacity`/`fill_opacity` to fixed baseline
values, and a Layout-type control procedure hides all five fields (`border_color`/`border_width`/
`border_opacity`/`fill_color`/`fill_opacity`) on the form for the rest of that row's life. Don't spend
effort choosing marker colors — they're discarded. A marker's actual visual appearance is driven by
the **domain element's own `icon`/`icon_id` field** (on `elemnt`, the same row `map_data_mapping_id`
must match), not by anything on `map_data_mapping` itself. Only `line`/`polygon`/`rectangle`/`circle`
rows actually use the color/width/opacity fields below.

**The built-in icon catalog is not reliably queryable through a metadata-driven connector** — a
model's `icon`/`icon.icon_look_up` entity can return zero rows even when icons are visibly in use
elsewhere in that same model. This is now explained by `thinkwise_software_factory_icons`: nearly every
icon-bearing entity, `elemnt` included, actually carries **two** independent mechanisms — an `icon_id`
foreign key into the shared repository, and a local `icon`/`icon_data` file uploaded straight onto the
row. A model whose markers were all styled via the local upload path (never entered into the shared
`icon` repository) will show icons rendering fine in the app while the `icon` entity itself sits empty.
Don't try to enumerate or guess a valid `icon_id` from a possibly-empty repository query. **If the user
hasn't specified which icon a marker should use, ask** (or hand them the picker) — per
`thinkwise_software_factory_mcp_base`'s "Ask, don't default" convention, and per
`thinkwise_software_factory_icons`'s own rule to ask rather than guess when no confident match exists —
rather than choosing one unilaterally.

### Decoding/encoding `border_color` and `fill_color` (non-marker geometric types only)

These are stored as a **signed 32-bit integer holding an ARGB value**, alpha in the high byte:

```
unsigned = value if value >= 0 else value + 2**32
alpha = (unsigned >> 24) & 0xFF
red   = (unsigned >> 16) & 0xFF
green = (unsigned >> 8)  & 0xFF
blue  =  unsigned        & 0xFF
```

To encode a desired ARGB back into the field, do the reverse and re-sign if the unsigned value exceeds
`2**31 - 1`:

```
unsigned = (alpha << 24) | (red << 16) | (green << 8) | blue
value = unsigned if unsigned < 2**31 else unsigned - 2**32
```

Verified worked example from a production `map_data_mapping` row styling a route line: field value
`-16776961` → unsigned `4278190335` → `0xFF0000FF` → alpha `FF` (opaque), red `00`, green `00`, blue
`FF` — solid opaque blue. Get this formula wrong and a control procedure or staged write silently
produces the wrong color (e.g. transparent, or a completely different hue) with no error at any step.

A single table's map_data_mapping rows can mix geometric types freely under one domain — verified:
one table used a 4-element domain where three elements were markers (`origin_marker`,
`destination_marker`, `waypoint_marker`) and the fourth was a `line` (the calculated route path),
letting one map render both point and path geometry from the same coordinate column.

**Legend/layer-ordering caveat**: Community reports note inconsistencies between how `map_data_mapping`
elements are ordered/sequenced in the Software Factory vs. how they render in a running app, and that
`show_map_data_mapping` hasn't always behaved as expected across Windows GUI vs. Universal GUI —
verify actual rendered behavior on the target GUI and platform version rather than trusting the
modeled order_no/flag alone.

## Setting up a basic map — step by step

0. **Confirm scope with the user before creating anything** — which table needs the map, what
   geometry types it needs (marker/line/polygon/rectangle/circle, or a mix), the initial center/zoom,
   which tile provider(s) to use, and whether end-user drawing (Geometric Type Creation) is needed.
   This is `thinkwise_software_factory_mcp_base`'s "Shared conventions" — Confirm-before-mutate and
   Ask, don't default — applied to a map specifically: don't create the coordinate column, the `map`
   row, or anything else in the steps below until this is settled.
1. **Confirm the connector's actual domain keys** for maps / data model / control procedures
   (`search_capabilities` or `get_available_domains`) — don't assume `manage_maps` etc. literally.
2. **Get (or build) a CoordSets-producing column** on the target table — a calculated field per "The
   CoordSets JSON contract" above, checking `branch_rdbms_type` first if hand-writing the SQL.
3. **Create the `map` row** for the table: set `latitude_longitude_col_id` to that column,
   `data_mapping_col_id` to a domain-based column, `popup_col_id` to whatever should show on click,
   and `initial_latitude`/`initial_longitude`/`initial_zoom_level` from what the user confirmed in
   step 0 — if that wasn't pinned down yet, ask for the initial center/zoom now, or infer it from a
   known reference row already confirmed with the user, rather than picking an arbitrary default.
4. **Add at least one `map_base_layer`** with a working tile URL template, a zoom range that matches
   what the provider actually serves, and (for most public providers) `attribution_html`. If the tile
   provider(s) weren't already confirmed in step 0, ask which to use rather than picking one
   unilaterally.
5. **Add one `map_data_mapping` row per element** of the domain used in step 3's `data_mapping_col_id`
   — pick `geometric_type` per element; for `line`/`polygon`/`rectangle`/`circle` elements, also set
   border/fill styling, decoding/encoding colors per the formula below; for `marker` elements, set an
   `icon` on the element instead (see the marker exception above) and leave color fields untouched.
   **API limitation, verified live: `map_data_mapping` could not be created or edited through a
   metadata-driven staging API in a verified session.** Its key spans two independent "parents" at
   once (`tab_id`, reached via `map`; `dom_id`/`elemnt_id`, reached via the domain's elements), and
   every path tried failed in a different way: staging under `map` or `tab` as parent reported no
   detail navigation to this entity; staging under `dom` as parent reported no such parent at all for
   this entity; a plain add with no parent left every key field permanently read-only, so even
   supplying the full compound key as ordinary fields (the usual fallback for this shape of weak
   entity — see `thinkwise_datamodeling_guidelines`) never became possible; and its own `_overview`
   sibling entity (which does resolve a detail navigation from `map`, and confirms the
   `map_data_mapping_id`-must-equal-an-`elemnt_id` contract via a `lookup_map_data_mapping_id → elemnt`
   navigation) was rejected outright on commit. Don't spend a session's budget chasing this — treat it
   as a manual step in the Software Factory's own UI by default, and only revisit if a specific
   connector's domain metadata is confirmed to expose it differently.
6. **Verify the eligible domain elements' `elemnt_id`s exactly match the `map_data_mapping_id`s
   created** — a mismatch silently leaves some rows unstyled rather than erroring.
7. Only if the table needs end-user drawing: wire the relevant `*_location_task_parmtr_id` field(s)
   on `map` — see "Drawable shapes" next.
8. Only if coordinates come from an external service rather than user input or plain columns: build
   the web-connection/process-flow/control-procedure pipeline — see "External-API pattern" below.

## Drawable shapes (Geometric Type Creation)

Drawing is the mirror image of data mapping: instead of a column feeding the map, the map feeds a
**task**. Since the map component's shape-drawing tools shipped, users can draw markers, circles,
rectangles, lines and polygons directly on the canvas; each draw action executes the task wired into
the matching `map.*_location_task_parmtr_id` field, passing the drawn geometry as that parameter's
value.

A circle's captured JSON carries one extra key beyond a plain CoordSets ring — `"Radius"`, in meters:

```json
{
  "CoordSets": [ [ { "Lat": 52.20839876100734, "Lon": 5.97939399968484 } ] ],
  "Radius": 63.19832731146133
}
```

Setup steps:

1. **Create a task** whose parameters are the target row's primary key column(s), plus one parameter
   (satisfying the `only_alphanumeric` eligibility above) that will receive the drawn geometry as
   CoordSets JSON — for circles, the receiving control procedure also needs the radius, either as a
   second parameter or parsed out of the same JSON payload depending on how the connector surfaces it.
   (This task, like any new task, starts with a bracket-placeholder translation — see
   `thinkwise_software_factory_translation_objects` to give it a real label before it reaches an end
   user's toolbar.)
2. **Assign the task as a table task** on the same table the map is defined on, mapping the
   primary-key parameters to their columns (`use_primary_key` on the table-task-creation task, or
   explicit `col_id_n`/`task_parmtr_id_n` pairs).
3. **Write the control procedure** that saves the incoming JSON onto the row — a plain `update`,
   verified pattern:

   ```sql
   -- task parameters: company_id, company_address_id (PK), geofence_coordset (task input, the JSON above)
   update company_address
   set geofence_coordset = @geofence_coordset,
       geofence_data_mapping_type = 3
   where company_id = @company_id
   and company_address_id = @company_address_id
   ```

4. **Wire it into the map**: set the relevant `map.*_location_task_parmtr_id` field (the "Geometric
   Type Creation" setting) to the task parameter created in step 1.

**Known platform gap**: no built-in surface-area or length calculation for a drawn shape (a circle
shows its radius on hover; an irregular polygon does not get an area). If a use case needs it (e.g. a
farm field's area in square meters), compute it from the `CoordSets` coordinates yourself — this is an
open, voted Community idea, not yet shipped; check its status before building a custom workaround.

**Circle sizing is in screen pixels for the rendered marker itself**, distinct from the geographic
`Radius` value captured on draw — the drawing tool's on-canvas circle uses Leaflet's `CircleMarker`,
whose own radius option is pixel-based and does not scale with zoom the way the captured `Radius`
(meters) does. Don't conflate the two when styling vs. when processing captured data.

## Tab-variant overrides

Every one of the four map entities has a `tab_variant_*` counterpart that overrides the table-level
default for one specific tab variant, without redefining the map from scratch:

| Entity | Overrides |
|---|---|
| `tab_variant_map` | Center/zoom and column wiring, per variant |
| `tab_variant_map_base_layer` | Base layer set, per variant |
| `tab_variant_map_overlay` | Overlay set, per variant |
| `tab_variant_map_data_mapping` | Styling, per variant |

Use this when the same table's map needs to look or behave differently depending on which screen opens
it (a compact dashboard summary map vs. a full detail-screen map), without maintaining two physical
map definitions. Each has a matching `_overview` read entity and a `task_reset_tab_variant_map*_overview`
bound task (`task_reset_tab_variant_map_overview`, `task_reset_tab_variant_map_base_layer_overview`,
`task_reset_tab_variant_map_data_mapping_overview`, `task_reset_tab_variant_map_overlay_overview`) to
revert a variant's override back to the table-level default in one call, rather than deleting override
rows individually.

## External-API pattern: geocoding and route calculation

For populating a map's coordinates from an external service instead of user input — a table task
(dummy trigger) → process flow → `web_connection` call → task → control procedure that parses the
response and writes CoordSets, including the verified `process_action` sequence, `web_connection`/
`web_connection_endpoint` field reference, and the full openjson/PostgreSQL JSON-parsing SQL examples —
read `references/external_api_pattern.md`.

## Bound-task quick reference

Exact task names verified on the map entities — use these rather than guessing when a connector needs
to mutate/inspect map configuration programmatically:

| Entity | Bound tasks |
|---|---|
| `map` | `task_delete_map`, `task_show_history`, `task_unlink_generated_object` |
| `map_base_layer` | `task_copy_map_base_layer`, `task_delete_map_base_layer`, `task_rename_map_base_layer`, `task_show_history`, `task_unlink_generated_object` |
| `map_overlay` | `task_copy_map_overlay`, `task_delete_map_overlay`, `task_rename_map_overlay`, `task_show_history`, `task_unlink_generated_object` |
| `map_data_mapping` | `task_show_history`, `task_unlink_generated_object` (no copy/rename — delete and re-add to change its key) |
| `tab_variant_map` / `_base_layer` / `_overlay` / `_data_mapping` | `task_reset_tab_variant_map_overview`, `task_reset_tab_variant_map_base_layer_overview`, `task_reset_tab_variant_map_overlay_overview`, `task_reset_tab_variant_map_data_mapping_overview` — revert an override back to the table-level default |

`task_copy_map_base_layer` / `task_copy_map_overlay` take `from_tab_id`/`from_map_base_layer_id`/
`to_tab_id`/`to_map_base_layer_id` (or the overlay equivalents) — useful for cloning a working tile
setup onto a new table rather than re-typing URIs and zoom ranges.

## Known pitfalls (Community-verified)

- **Pagination hides markers, not a hard limit.** The map only renders rows on the currently loaded
  page — if locations are silently missing, raise the table's/task's page size before assuming a
  platform cap.
- **First load can center on (0, 0) ("Null Island").** A map showing a route has been reported to
  open there on the very first render, then auto-fit correctly on every later open — a confirmed bug,
  not a modeling error. Setting an explicit `initial_latitude`/`initial_longitude` may mask it for
  small extents.
- **Max zoom is the tileset's limit, not the GUI's** — see `map_base_layer`/`map_overlay` field
  reference above.
- **No bearer/OAuth2 tile layers directly** — see the same section; use a proxy.
- **Data-mapping legend order/visibility toggle has had cross-GUI inconsistencies** — verify on the
  actual target GUI and version rather than trusting the modeled order alone.
- **Circle drawing size is pixel-based on-canvas**, distinct from the geometric `Radius` captured in
  the drawn shape's JSON — don't conflate visual size with the meters value.
- **No built-in shape area/length calculation** — compute from `CoordSets` yourself if needed; watch
  the relevant Community idea for platform support.
- **A `marker` row's color/width/opacity fields are hidden and nulled by the platform itself** — style
  a marker via its domain element's `icon` instead, not via `map_data_mapping`'s color fields. See the
  marker exception under the `map_data_mapping` field reference above.
- **`Lon`/`Lat` transposed in a CoordSets expression produces no error, just a silently wrong pin
  location** — verify against a known real-world coordinate, not just that the map renders something.

## Pre-flight checklist

- Confirm the connector's actual domain keys for maps / data model / control procedures / web
  connections / process flows before assuming `manage_maps`-style names hold.
- `map.latitude_longitude_col_id` must point at a column producing valid CoordSets JSON — verify the
  dialect (`branch_rdbms_type`) before hand-writing any SQL that builds or parses it.
- `map.data_mapping_col_id`'s domain element IDs must exactly match the `map_data_mapping_id`s
  created for that `dom_id` — a mismatch silently leaves rows unstyled, it does not error.
- For `marker` rows, don't set `border_color`/`fill_color`/etc. at all — the platform hides and nulls
  them; set the domain element's `icon` instead. Decode/encode `border_color`/`fill_color` as
  signed-32-bit ARGB using the ⟨alpha, red, green, blue⟩ formula above only for the other four
  geometric types.
- Expect `map_data_mapping` itself to be unstageable through a metadata-driven write API — confirm the
  connector's own behavior before assuming otherwise, and default to treating it as a manual step.
- `map_base_layer`/`map_overlay` `max_zoom_level` must match what the tile provider actually serves,
  not an aspirational value.
- Bearer/OAuth2-secured **tile layers** need a proxy; bearer/OAuth2-secured **API calls** for
  geocoding/routing are natively supported via `web_connection.authentication_type`.
- The five `map.*_location_task_parmtr_id` fields only accept task parameters passing the
  `only_alphanumeric` eligibility filter — confirm before assigning.
- The dummy-trigger-task pattern (`task_type_id = 'DUMMY'`, all parameters hidden) is expected and
  correct for a process-flow-driven map data source — don't add UI-visible fields or dynamic-model
  code to it; the logic belongs in the process flow and its control procedures.
- Verify the actual process-variable name carrying a web connection's raw response body
  (`@response_http_content` in the verified reference model) against the connector's own process-flow
  variable list — don't port it blindly.
- `openjson`/`cross apply`/`for json path` (SQL Server) vs. `jsonb_*`/`json_agg`/`json_build_object`
  (PostgreSQL) — check `branch_rdbms_type` before writing any JSON parsing/building control procedure,
  same as every other control procedure.
- Delete-and-reinsert detail/map rows on recalculation rather than diff-and-update, and don't forget a
  cascade-delete handler on the parent.
- For anything not map-specific (naming, domain reuse, reference direction, control-procedure
  generation mechanics) defer to `thinkwise_datamodeling_guidelines` and
  `thinkwise_software_factory_create_control_procedures` rather than duplicating those rules here.
