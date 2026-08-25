# External-API pattern: geocoding and route calculation

The pattern for populating a map's coordinates from an external service (geocoding an address, calling
a routing API) rather than user input, verified end-to-end against a working reference implementation.

## Architecture

```
table task (button)  →  process flow  →  web connection call  →  task  →  control procedure
   "get_route"         "get_and_save_    (external API,          "save_    (parses response,
   on `route`            route"           e.g. Google              route"    writes CoordSets
                                           Directions API)                    into the table)
```

## The trigger task is a "dummy" task

The button the user clicks is a **table task** (`tab_task`) bound to a **task** with `task_type_id =
'DUMMY'` — verified: no server-side dynamic-model code of its own, every parameter's `type_of_col =
hidden`, `task_input = true`. Its only job is to exist as a named, parameterized entry point a process
flow can invoke (`process_action_type = execute_tab_task`, enum value `6`) and carry the row's current
field values forward into the flow. All real logic lives in the process flow and the control
procedures it calls, not in the task itself.

## The process flow

Built from `process_action` rows (keyed by `(model_id, branch_id, process_flow_id,
process_action_id)`), each an `Edm.Int32` enum `process_action_type`. The ones this pattern uses,
verified from a live model:

| `process_action_type` | Value | Role in this pattern |
|---|---|---|
| `start` | 98 | Flow entry point |
| `execute_tab_task` | 6 | Fires the dummy trigger task, carrying the row's current values |
| `web_connection` | 602 | Calls the external API via a `web_connection_id`/`web_connection_endpoint_id` pair |
| `execute_task` | 60 | Runs the "save" task whose control procedure parses the response |
| `refresh_tab` | 8 | Refreshes the screen so the newly written map rows render |
| `stop` | 99 | Flow exit point |
| `add_record` / `edit_record` | 3 / 4 | Alternate entry points feeding the same flow when the record is being created/edited, not just when the trigger button is pressed |

## The web connection

`web_connection` (keyed by `(model_id, branch_id, web_connection_id)`) holds the base URL and
authentication; `web_connection_endpoint` holds one path/method per API operation;
`web_connection_endpoint_query_string_parmtr` maps query-string keys to endpoint parameter
placeholders (`{param_name}` substitution syntax, verified); `web_connection_endpoint_output_parmtr`
declares where in the response to read a named output from (`endpoint_output_mapping` enum:
`http_status_code` = 0, `response_body` = 1, `response_header` = 2).

Verified example — a `google` web connection, API-key auth carried as a query parameter
(`authentication_type = api_key`, `api_key_parmtr_location = query_parmtr`), two `GET`
(`type_of_body = body_type_none`) endpoints:

| Endpoint | Path | Query parameters |
|---|---|---|
| `geocode` | `maps/api/geocode/json` | `address={address}` |
| `directions` | `maps/api/directions/json` | `origin={origin_lat},{origin_lon}` · `destination={dest_lat},{dest_lon}` · `mode={travel_mode}` · `units={route_unit}` · `waypoints={waypoint}` |

Build any query-string value that needs conditional logic (e.g. an optional waypoint that must vanish
entirely, not just go blank, when unset) with a small control procedure run before the call fires:

```sql
-- Check if the latitude is empty, if not concat, if so then leave waypoint empty.
set @waypoint = iif(@waypoint_latitude is null, null, concat(@waypoint_latitude, ',', @waypoint_longitude))
```

**Raw response body variable — verify, don't assume the name.** The control procedures that run after
a `web_connection` process action, in the verified reference model, read the raw response JSON via a
variable named `@response_http_content` — this reads like a platform-provided process variable holding
the immediately-preceding web-connection call's response body, distinct from any explicitly named
`web_connection_endpoint_output_parmtr` (which existed, named `result`, mapped from `response_body`,
but was not what the control procedures actually referenced). Confirm the exact variable/parameter
name your connector's process-flow variable list actually exposes before writing a parsing control
procedure — don't port `@response_http_content` blindly into a different model without checking.

**Bearer/OAuth2 web-service auth**: `web_connection.authentication_type` supports `oauth` (5) and
`bearer_token` (2) directly for calling out to external APIs (unlike the base-layer/overlay tile case
above, which cannot) — pick the type matching the target API's actual requirement rather than
defaulting to `api_key`.

## The parsing/writing control procedure

The response-handling control procedure typically needs to (a) parse a nested JSON response and (b)
build one or more CoordSets strings from the parsed data. On SQL Server, this is `openjson(...) cross
apply` chains for parsing, and either string concatenation or `for json path` for building — verified,
condensed pattern from a real "save calculated route" control procedure:

```sql
-- Parse: pull distance/duration out of a nested routes → legs → steps response
select s.distance_value, s.distance_text, s.duration_value, s.duration_text
from openjson(@response_http_content)
     with (status varchar(100) '$.status', routes nvarchar(max) '$.routes' as json) r
cross apply openjson(r.routes) with (legs nvarchar(max) '$.legs' as json) l
cross apply openjson(l.legs)
     with (steps nvarchar(max) '$.steps' as json
          ,distance_value int '$.distance.value', distance_text varchar(100) '$.distance.text'
          ,duration_value int '$.duration.value', duration_text varchar(100) '$.duration.text') s
where r.status = 'OK'

-- Build: a single-point CoordSets for a marker, from plain columns
select '{"CoordSets":[' + (select r.origin_latitude as Lat, r.origin_longitude as Lon
                            from route r where r.route_id = @route_id
                            for json path) + ']}'

-- Build: a multi-point CoordSets for a line, from every parsed step's start coordinate
select '{"CoordSets":[' + (
    select ss.start_location_lat as Lat, ss.start_location_lng as Lon
    from openjson(@response_http_content)
         with (status varchar(100) '$.status', routes nvarchar(max) '$.routes' as json) r
    cross apply openjson(r.routes) with (legs nvarchar(max) '$.legs' as json) l
    cross apply openjson(l.legs) with (steps nvarchar(max) '$.steps' as json) s
    cross apply openjson(s.steps)
         with (start_location_lat numeric(9,6) '$.start_location.lat'
              ,start_location_lng numeric(9,6) '$.start_location.lng') ss
    where r.status = 'OK'
    for json path
) + ']}'
```

Delete-and-reinsert is the verified pattern for re-running a calculation (route recalculated with a
new waypoint, etc.) rather than trying to diff and update in place — simpler and correct given the
detail rows have no independent identity worth preserving:

```sql
delete from route_details where route_id = @route_id
-- …then re-insert one row per marker + one row for the line, as above
```

**Check `branch_rdbms_type` before writing any of this.** See the
`thinkwise_software_factory_create_control_procedures` skill for the full dialect-check walkthrough;
the essential rule is: never assume SQL Server syntax works on PostgreSQL, or vice versa — verify first.
Specifically for this JSON parsing/building pattern: `openjson`/`cross apply`/`for json path` are SQL
Server-specific; on PostgreSQL the equivalent parsing uses `jsonb_array_elements`/
`jsonb_extract_path_text` (or `->`/`->>` operators) in place of `openjson … with (...)`, and building an
array of objects uses `json_agg(json_build_object(...))` in place of string-concatenation `for json
path`. Don't assume the SQL Server shape above ports as-is — translate per the target dialect.

**Defaulting related fields from a selected row** (e.g. filling a route's own lat/lon fields the
moment an origin/destination address is picked) is a default-value control procedure keyed off
`@cursor_from_col_id`, template-parameterized once and reused for multiple symmetric fields via a
placeholder token substituted at generation time:

```sql
-- template placeholder: [origin_or_destination], reused for both "origin" and "destination"
if (@cursor_from_col_id is null or @cursor_from_col_id = '[origin_or_destination]_address_id')
begin
    if @[origin_or_destination]_address_id is not null
    begin
        select @[origin_or_destination]_latitude = a.latitude
            , @[origin_or_destination]_longitude = a.longitude
        from address a
        where a.address_id = @[origin_or_destination]_address_id
    end
    else
    begin
        set @[origin_or_destination]_latitude = null
        set @[origin_or_destination]_longitude = null
    end
end
```

**Cascade-delete detail rows** when the parent is deleted with a plain delete-handler control
procedure — nothing map-specific here, just don't forget it (an orphaned `route_details` row with no
parent silently stops rendering rather than erroring):

```sql
delete rd from route_details rd where rd.route_id = @route_id
```
