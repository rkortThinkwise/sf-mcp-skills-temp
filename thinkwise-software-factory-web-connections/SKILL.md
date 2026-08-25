---
name: thinkwise-software-factory-web-connections
description: Reference guide for creating and using Web Connections and OAuth servers in a Thinkwise Software Factory model, the preferred mechanism for calling external HTTP/REST APIs. Covers authentication, endpoint and parameter setup, response parsing, and wiring a connection into a process flow. Use whenever an MCP connector with Software Factory access creates, inspects, or troubleshoots a web connection or OAuth server, in place of a one-off http_connector process action.
---

# Web Connections in the Thinkwise Software Factory

A **Web Connection** is a modeled, reusable definition of an external HTTP(S) API: one base URL and
authentication scheme (`web_connection`), with one or more named **endpoints** underneath it
(`web_connection_endpoint` — method, path, body). It replaces the legacy `http_connector` process
action, which re-enters URL/method/headers/auth on every single process action with no reuse, no
override mechanism, and no built-in response parsing.

Follow the connector's standard discovery→act flow; never guess entity/field/enum names — confirm
them via `get_entity_definition` first. **Two domains are involved, and mixing them up is the most
likely mistake:**

- **`sf/manage_webconnections`** — the connection/endpoint object graph itself: `web_connection`,
  `web_connection_endpoint` and every child (parameters, headers, query strings, form fields, output
  parameters), plus `oauth_server`. Build the connection here.
- **`sf/manage_process_flows`** — where the `web_connection` process action lives, plus its
  input/output *binding* entities (`process_action_web_connection_endpoint_parmtr_input_parmtr`,
  `process_action_web_connection_parmtr_input_parmtr`,
  `process_action_modeler_web_connection_endpoint_output`). Wire the already-built connection into a
  flow here — see `thinkwise_software_factory_process_flows`, which this skill complements rather than
  duplicates.

Try these domain keys directly first, and only escalate to `search_capabilities`/
`get_available_domains` on an `entity_set_not_found`/`domain_not_found`-style rejection rather than
re-discovering a domain that already resolved earlier this session.

> **Correction to an earlier finding.** An earlier pass at the process-flows skill inspected
> `web_connection`/`web_connection_endpoint` only through `sf/manage_process_flows` (where a process
> action merely *references* a connection/endpoint by id) and concluded the objects exposed "almost no
> writable configuration." That conclusion was scoped to the wrong domain. Read directly from
> `sf/manage_webconnections`, both entities carry their full configuration — `base_url`,
> `authentication_type`, credentials, `endpoint_http_method`, `endpoint_path`, `type_of_body`,
> `endpoint_body`, etc. (see field tables below). Building a web connection from scratch is fully
> modeled through this API; it does not require the Software Factory's own UI.

## Entity map

```
web_connection                                    (sf/manage_webconnections)
├─ web_connection_parmtr                           connection-wide {param}, reused across endpoints
├─ web_connection_configuration(_overview)          per runtime-configuration / per-app override
└─ web_connection_endpoint                          one HTTP call shape
   ├─ web_connection_endpoint_parmtr                 endpoint-scoped input {param}
   ├─ web_connection_endpoint_query_string_parmtr    name/value, omit-when-empty
   ├─ web_connection_endpoint_request_header         name/value, omit-when-empty
   ├─ web_connection_endpoint_form_field              multipart/form fields, incl. file uploads
   └─ web_connection_endpoint_output_parmtr           value extracted from the response

oauth_server                                       (sf/manage_webconnections, branch-scoped, standalone)

process_action (process_action_type = web_connection, 602)     (sf/manage_process_flows)
├─ process_action_web_connection_endpoint_parmtr_input_parmtr   endpoint input → process variable / literal
├─ process_action_web_connection_parmtr_input_parmtr            connection input → process variable / literal
└─ process_action_modeler_web_connection_endpoint_output        endpoint output → process variable
```

Every entity under `web_connection` is keyed by `(model_id, branch_id, web_connection_id, …)`, cascading
one more key segment per nesting level (e.g. `web_connection_endpoint_parmtr` is keyed by
`(model_id, branch_id, web_connection_id, web_connection_endpoint_id, web_connection_endpoint_parmtr_id)`).
All of them carry `_model_history` shadow entities and a `task_show_history` bound task.
`oauth_server` sits at branch level, independent of any one connection — a single OAuth server can be
referenced by several web connections' `oauth_server_id`.

**Every child entity is a weak entity under its parent** — stage with `parent_entity_set` set to the
immediate parent (`web_connection` for `web_connection_endpoint`/`web_connection_parmtr`;
`web_connection_endpoint` for everything under an endpoint) and `parent_key` set to the parent's full
composite key, the same pattern used throughout the Software Factory API (see
`thinkwise_software_factory_process_flows` for the general weak-entity staging convention).

## Golden rule: confirm the integration design before building

A web connection encodes decisions that are awkward to unwind once endpoints and process flows
depend on them — which auth mechanism the target API demands, where its credentials live, and how
each response gets parsed. Apply the mcp_base skill's "Shared conventions" (confirm-before-mutate,
ask-don't-default) at the grain of this specific integration, before step 1 of Build order below.
Confirm with the user:

- **The external API and its base URL per environment** — which service, and whether dev/test/acc/prod
  need different `base_url` values (see [Overrides](#overrides--encryption)) or one value that's fine
  everywhere.
- **`authentication_type`, and where its credentials live** — stored on the base connection, deferred
  to the process flow (`use_deferred_credentials`), or stored encrypted (`encryption_used`). These are
  three distinct, largely mutually-exclusive mechanisms, not defaults to pick silently — see the
  `use_deferred_credentials` note under [Overrides & encryption](#overrides--encryption).
- **The endpoint list and each endpoint's parsing strategy** — which calls are actually needed, and
  whether each response is parsed with JSONPath, XPath, or a regex (see
  [Output parameters](#5--output-parameters--parsing-the-response)).

Get the user's explicit sign-off on this list before starting Build order step 1 — these choices are
the design, not implementation mechanics.

## Build order

1. `web_connection` — base URL, auth type, credentials (or `oauth_server_id` if auth type = OAuth)
2. `oauth_server`, only if auth type = OAuth and one doesn't already exist for this API — see
   `references/oauth.md`'s API-gap note before assuming this is fully configurable via this API
3. `web_connection_parmtr`, only if a value needs to be shared across multiple endpoints (e.g. a
   tenant id used in several paths)
4. `web_connection_endpoint` rows — one per distinct API call
5. Per endpoint: `web_connection_endpoint_parmtr`, `_query_string_parmtr`, `_request_header`,
   `_form_field` (multipart only), `_output_parmtr`
6. Wire into a process flow: a `web_connection`-type `process_action`, then the input/output binding
   rows — see `thinkwise_software_factory_process_flows`
7. Optional: runtime-configuration / IAM per-application overrides, encrypted credential storage

## 1 · The web connection

Entity `web_connection` — one per external service/base URL.

| Field | Type | Notes |
|---|---|---|
| `web_connection_id` | key | Object name — see [Naming](#naming). |
| `web_connection_description` | string | Display label. |
| `base_url` | string | Scheme + host + fixed path prefix, e.g. `https://api.hubspot.com`. Overridable per environment — see [Overrides](#overrides--encryption). |
| `authentication_type` | `Edm.Byte` enum | `none`=0 · `basic_credentials`=1 · `bearer_token`=2 · `api_key`=3 · `managed_identity`=4 · `oauth`=5. Patch the raw integer, not the string label — same gotcha as `process_action_type` in the process-flows skill. |
| `user_name` / `password` | string | Auth type = Basic credentials. |
| `bearer_token` | string | Auth type = Bearer token. Pair with an upstream OAuth process action if the token needs periodic refresh — see [Wiring into a process flow](#wiring-into-a-process-flow). |
| `api_key_name` / `api_key_value` | string | Auth type = API key. |
| `api_key_parmtr_location` | `Edm.Byte` enum | `header_parmtr`=0 · `query_parmtr`=1 — where the key is injected. |
| `oauth_server_id` | ref → `oauth_server` | Auth type = OAuth. |
| `use_deferred_credentials` | flag | Push authentication into the *process flow* instead of storing it on the connection (supplied via web connection parameters at run time). **Once set, auth settings can no longer be overridden per application in IAM or per runtime configuration** — pick one mechanism, not both. |
| `client_certificate_file` | file (`.pfx`) | Mutual-TLS / client certificate auth — combines with any `authentication_type`, e.g. OAuth client-credentials *plus* mTLS. See the mTLS example in `references/worked_examples.md`. |
| `client_certificate_password` | string | Password protecting the `.pfx`. |
| `encryption_used` | flag | Store the credential fields encrypted (3-tier / Universal GUI branch-level encryption feature) — see [Overrides](#overrides--encryption). |

If the target API's required `authentication_type` or its credential-encryption policy (`encryption_used`)
isn't specified by the user, **ask** rather than defaulting to `none`/unencrypted — see
[Golden rule](#golden-rule-confirm-the-integration-design-before-building).

**Bound tasks:** `task_copy_web_connection`, `task_rename_web_connection`, `task_delete_web_connection`,
`task_encrypt_web_connection_key` (writes an encrypted credential value for a given
`authentication_type`), `task_reset_encrypt_web_connection_key`, `task_unlink_generated_object`,
`task_show_history`. Renaming cascades to every usage site automatically, same as elsewhere in the
Software Factory.

A connection can itself be model-generated (`generated_by_control_proc_id` set) — this is how
Thinkstore integration scripts scaffold connections programmatically; don't be surprised to find one
already wired up with this field populated.

**Not yet live-verified through this connector:** which of the above fields the write API treats as
strictly mandatory at creation vs. safely omittable. Default to setting several fields together in one
`stage_resource` call, per the tool's own recommended flow ("set the fields you know via properties in
this call to save a round-trip") — a multi-field drop is a real, confirmed failure mode on some entities
elsewhere in this skill set (see `thinkwise_software_factory_mcp_base`'s hazard note), but it isn't a
default assumption to apply pre-emptively here without having actually observed it on `web_connection`.
Re-read the `fields` block after the combined call to confirm every property landed; only fall back to
one-field-at-a-time isolation if a drop is actually observed on this entity.

## 2 · Authentication types

| Value | Name | Fields used | Notes |
|---|---|---|---|
| 0 | `none` | — | Public/unauthenticated API. |
| 1 | `basic_credentials` | `user_name`, `password` | Sent as `Authorization: Basic`. |
| 2 | `bearer_token` | `bearer_token` | Sent as `Authorization: Bearer <token>`. |
| 3 | `api_key` | `api_key_name`, `api_key_value`, `api_key_parmtr_location` | Injected as a header or query string param per `api_key_parmtr_location`. |
| 4 | `managed_identity` | — | Cloud-platform managed identity, no stored secret. Has been an active community feature request specifically for the *HTTP* connector — confirm current behavior for the target cloud (Azure) against the live model/version before depending on it for a web connection. |
| 5 | `oauth` | `oauth_server_id` | Delegates token acquisition to a configured OAuth server — see below. |

A client certificate (`client_certificate_file`/`_password`) is independent of `authentication_type`
and layers on top of it; it is not itself a 7th auth-type value.

## OAuth servers

OAuth server configuration (`oauth_server`, its known API gap) and the three OAuth process actions
(Server Login, User Login, Refresh Token — types, inputs/outputs, status codes) are covered in
`references/oauth.md`. Load it before configuring `authentication_type = oauth` on a connection or
wiring an OAuth process action into a flow.

## 3 · Endpoints

Entity `web_connection_endpoint` — one HTTP call shape under a connection.

| Field | Type | Notes |
|---|---|---|
| `web_connection_endpoint_id` | key | Object name, e.g. `get_contacts`. |
| `endpoint_http_method` | string enum | `GET` · `POST` · `PUT` · `PATCH` · `DELETE` · `HEAD` · `OPTIONS` · `TRACE` — values are the method names themselves, not numeric codes. |
| `endpoint_path` | string | Appended to `base_url`. `{token}` segments become endpoint input parameters automatically, e.g. `{table}({key})/appl.preview_{file_column}`. |
| `type_of_body` | `Edm.Byte` enum | `multipart_form`=0 · `form_url_encoded`=1 · `json`=2 · `xml`=3 · `yaml`=4 · `plain_text`=5 · `other`=6 · `none`=9. |
| `content_type_body` | string | `Content-Type` header sent — auto-suggested from `type_of_body`, overridable. |
| `endpoint_body` | string | Literal request-body template with `{param}` substitutions. |
| `endpoint_full_url` | computed, read-only | Preview of `base_url` + `endpoint_path`. |

**Query string parameters** (`web_connection_endpoint_query_string_parmtr`, key
`query_string_parmtr_name`): `query_string_parmtr_value` plus `omit_when_empty` to drop the param
entirely rather than send it blank.

**Request headers** (`web_connection_endpoint_request_header`, key `request_header_name`):
`request_header_value` plus `omit_when_empty`.

**Form fields** — multipart or form-url-encoded bodies (`web_connection_endpoint_form_field`, key
`form_field_name` + a generated `web_connection_endpoint_form_field_id`): `form_field_value`,
`omit_when_empty`, `derive_content_type_form_field` (infer content type from the file name),
`content_type_form_field` (explicit MIME type), `file_name` (parameterizable, e.g. `{file_name}`, for
binary uploads).

**Bound tasks:** `task_copy_web_connection_endpoint`, `task_rename_web_connection_endpoint`,
`task_delete_web_connection_endpoint`, plus rename/delete pairs for endpoint input parameters and
output parameters.

## 4 · Input parameters &amp; pre-processing

Two parameter scopes share the same shape: `web_connection_parmtr` (connection-wide — reusable across
every endpoint, e.g. a tenant id) and `web_connection_endpoint_parmtr` (single endpoint only).
Reference either with `{parameter_name}` anywhere textual — path, query string value, header value,
form field value, or body template. Renaming (`task_rename_web_connection_parmtr` /
`_endpoint_parmtr`) rewrites every usage site automatically, tracked in the read-only
`*_parmtr_usage` entities.

- `default_value` — constant fallback if the process flow doesn't supply one.
- `*_pre_processing` — how the raw value is escaped/encoded before substitution:

| Value | Name | Effect |
|---|---|---|
| 0 | `pre_processing_auto` | Chosen automatically based on where the parameter is used (query string vs. JSON body vs. XML body, etc.). |
| 1 | `pre_processing_query_string_value` | URL-encode special characters. |
| 2 | `pre_processing_json` | Escape as a JSON string value (escapes `"`). |
| 3 | `pre_processing_xml` | Escape as an XML string value. |
| 4 | `pre_processing_base64` | Base64-encode the value. |
| 9 | `pre_processing_none` | Substitute verbatim — required when the parameter's value is itself a pre-built JSON fragment/array. |

> **Escaping gotcha, from a real migration.** A JSON-body parameter is auto-escaped by default, so
> dropping `{param}` straight into a body template usually fails with "non parsable body" unless the
> value is a plain scalar. If the value is already-serialized JSON (e.g. a nested array from
> `FOR JSON PATH`), keep the body template **static** — `{ "documents": {document_array} }` — and set
> that parameter's pre-processing to `pre_processing_none`. Splicing the whole pre-built object in as
> `{request_body}` with default escaping, or hand-writing the structure with per-field scalar
> substitutions inside it, both produced unparsable bodies in practice. See
> `references/worked_examples.md`.

## 5 · Output parameters — parsing the response

Entity `web_connection_endpoint_output_parmtr`.

| Field | Type | Notes |
|---|---|---|
| `endpoint_output_mapping` | `Edm.Byte` enum | `http_status_code`=0 · `response_body`=1 · `response_header`=2 — where the value is read from. |
| `endpoint_output_response_header` | string | Header name, when mapping = response header. |
| `base64_decode` | flag | Decode the extracted value from Base64 before returning it. |
| `json_path_expression` | string | JSONPath applied to the response body, e.g. `$.results..properties`. |
| `always_as_array` | flag | Force the JSONPath result to be wrapped as an array even for a single match — keeps downstream parsing consistent. |
| `xpath_expression` | string | XPath applied to an XML response body. |
| `regular_expression` | string | Regex applied to extract a substring — works on any text response, not just JSON/XML. |

Normally only one of JSONPath / XPath / Regex is set per output parameter, matching whatever format
the API actually returns (independent of `type_of_body`, which only governs the outgoing request
body). Beyond the endpoint's own output parameters, the `web_connection` process action also exposes a
generic **status code** (0 = success; negative values for send/parse/timeout failures — check the live
status-code translations in the model rather than assuming a fixed list) and the raw **HTTP status
code** (200, 404, 500 …) regardless of any JSONPath/XPath mapping.

## Wiring into a process flow

Covered in depth by `thinkwise_software_factory_process_flows`; summarized here for the parts specific
to a web connection.

Add a `process_action` of type **`web_connection`** (602) in `sf/manage_process_flows`, with
`web_connection_id` and `web_connection_endpoint_id` both mandatory. The modeler then pre-seeds one
binding row per input/output parameter that endpoint (and any connection-level parameters it uses)
exposes — same pre-seeded, edit-only pattern as every other action type's I/O (query the placeholder
row first, then `edit` it; `add` against these child entities is rejected).

**Inputs** — `process_action_web_connection_endpoint_parmtr_input_parmtr` (per endpoint parameter) and
`process_action_web_connection_parmtr_input_parmtr` (per connection-level parameter used by this
endpoint):

- `assignment_method` — `variable` (pull from a `process_variable_id`) or `literal_constant` (a fixed
  `constant_value` typed into the model).

**Outputs** — `process_action_modeler_web_connection_endpoint_output`: one row per endpoint output
parameter, each with a `process_variable_id` to receive the extracted value.

If the API needs a bearer token obtained via OAuth, precede the `web_connection` action with an
`oauth_server_login_connector`/`oauth_user_login_connector` action and bind its access-token output to
the web connection's `bearer_token` input (only reachable this way if `authentication_type` = Bearer
token with the token supplied at run time, or via `use_deferred_credentials`) — or set
`authentication_type = oauth` directly on the connection and let it manage the exchange itself via
`oauth_server_id`, when the OAuth server field gap noted in `references/oauth.md` doesn't block that.

**Migrating an existing `http_connector` action:** the domain exposes a bound enrichment task on
`process_action`, `task_enrichment_conv_http_connector_to_web_connection` ("Convert HTTP connector to
Web connection with AI") — try this before hand-porting URL/headers/body manually.

## Overrides &amp; encryption

Because credentials differ by environment, base URL and authentication settings can be overridden
without touching the model:

- **Runtime configuration** (Software Factory → Maintenance) — `web_connection_configuration` /
  `web_connection_configuration_overview`, keyed additionally by `runtime_configuration_id`, mirroring
  every field on the base connection. `task_switch_web_connection_type` changes the auth type for one
  configuration; `task_reset_web_connection_configuration` reverts it to inherit the base connection.
- **Per-application override in IAM** (Authorization → Applications) — same fields, scoped to one
  deployed application instead of a runtime configuration.

Credential fields on either the base connection or an override can be stored encrypted
(`encryption_used`) via `task_encrypt_web_connection_key`, which takes the target
`authentication_type` plus whichever of `password` / `bearer_token` / `api_key_value` /
`client_certificate_password` applies.

> **`use_deferred_credentials` opts out of both overrides.** If set, authentication is supplied by the
> process flow itself at run time instead of being stored on the object, and as a direct consequence it
> can no longer be overridden per application in IAM or per runtime configuration. Pick one mechanism.
> **Ask the user explicitly before setting it** rather than defaulting to it as a convenience — it
> forecloses both override mechanisms at once and is not easily reversible once process flows are
> built assuming auth arrives this way.

## Worked examples

Three full walkthroughs — reading IAM's user table via Indicium, HubSpot's nested JSONPath response,
and mTLS + OAuth client credentials for a bank API — are in `references/worked_examples.md`. Purely
illustrative; skip unless you want a concrete precedent for one of these patterns.

## Known gotchas

- **Multipart form data on GET may be silently ignored.** One community report against an external
  API found the web connector's multipart form body wasn't applied on a GET request (an unfiltered
  full list came back instead of the filtered one) even though the process flow monitor showed the
  parameters as sent correctly. If this is hit, consider a support ticket and/or falling back to
  `http_connector` for just that one call rather than the whole integration.
- **JSON body escaping is per-parameter, not per-request** — a "non parsable body" response almost
  always means a parameter's pre-processing mode doesn't match whether its value is a scalar or
  already-serialized JSON. See the escaping gotcha above.
- **`use_deferred_credentials` disables both override mechanisms** — don't combine it with an
  expectation of environment-specific secrets stored in the model.
- **`oauth_server`'s real configuration fields are not exposed through `sf/manage_webconnections`**
  (only the key and `generated_by_control_proc_id`) — verify before promising a fully API-driven OAuth
  server setup; see `references/oauth.md`.
- **Managed identity** as an auth type has mostly been discussed as an HTTP-connector feature request
  in the community — confirm it behaves as expected for the target cloud before depending on it here.

## Naming

No dedicated web-connection naming convention has been observed beyond general Thinkwise guidance
(`thinkwise_datamodeling_guidelines`): lowercase snake_case, no abbreviations. In practice, name the
connection for the external service (`hubspot`, `azure_ad`, `iam`) and each endpoint for the operation
it performs (`get_contacts`, `create_invoice`, `usr`) rather than after the HTTP verb or URL shape.

## Translation

`web_connection_description`, `web_connection_endpoint_description`, and the parameter/output
descriptions are translatable objects, same as any other model object — don't leave new ones in the
source language only. See `thinkwise_software_factory_translation_objects`.

## Pre-flight checklist

- Confirm domain keys per-connector rather than assuming `sf/manage_webconnections` /
  `sf/manage_process_flows` hold — but don't re-discover a domain that already resolved earlier this
  session.
- Build the connection graph in `sf/manage_webconnections` **before** touching the process-flow
  binding entities in `sf/manage_process_flows` — the process action can't bind to a connection or
  endpoint that doesn't exist yet.
- Patch `authentication_type`, `endpoint_http_method`, `type_of_body`, and every other byte/enum field
  as the raw numeric or exact string value from its enum list — not a guessed label.
- Match a body-template parameter's pre-processing mode to what its *value* actually is (scalar vs.
  already-serialized JSON/XML) — this is the single most common cause of an unparsable request body.
- Prefer `web_connection` over `http_connector` for any new integration — it's fully modeled through
  this API (base URL, auth, endpoints, parsing, overrides), reusable across every process flow that
  calls the same API, and overridable per environment without touching the model. Reach for
  `http_connector` only for a genuine one-off call that will never be reused, or as a documented
  fallback for a specific gotcha (e.g. the multipart-on-GET issue above).
- Before configuring OAuth: verify whether `oauth_server`'s client id/secret/endpoint fields are
  actually writable through this connector (the schema read shows only the key) rather than assuming
  the docs' field list is reachable via this API.
- Model output parameters (JSONPath/XPath/regex) rather than leaving response parsing to a downstream
  `extract_json` action or raw SQL — that's the whole point of preferring a web connection.
- Set `use_deferred_credentials` only when the flow-supplied-auth pattern is genuinely needed — it
  forecloses both the runtime-configuration and IAM per-application override mechanisms.
- New connection/endpoint descriptions are translatable objects — fill them in via
  `thinkwise_software_factory_translation_objects` before considering the connection finished.
