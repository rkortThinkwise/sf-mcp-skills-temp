# Worked examples

**Reading IAM's user table via Indicium** (community walkthrough):
1. `base_url` → the environment's Indicium IAM endpoint, e.g. `http://localhost/indicium/iam/iam/`.
2. Auth type per-API: OAuth where possible; Basic is acceptable if also IP-whitelisted; PATs exist on
   recent versions but aren't intended for server-to-server calls.
3. Endpoint with `endpoint_path = usr` — the built-in IAM view exposing all user rows.
4. Output parameter mapped to `response_body` to capture the raw JSON.
5. A (system) process flow calls the endpoint and parses the JSON output.
6. To promote across dev/test/acc/prod, change only `base_url` per environment (or per runtime
   configuration) — endpoints always resolve against it, nothing else in the model changes.

**HubSpot — nested results with JSONPath:** a list endpoint returning contacts nested under
`results[].properties` uses `json_path_expression: $.results..properties` with `always_as_array: true`
so the shape stays consistent even for a single hit.

**Sending a pre-built JSON array as the body:** see the pre-processing gotcha in the main
`SKILL.md` (Input parameters &amp; pre-processing section) — static body template
`{ "documents": {document_array} }`, parameter pre-processing set to `none`.

**mTLS + OAuth client credentials (bank API):** the OAuth Server Login connector doesn't support
certificates, so:
1. Merge separate `.cert`/`.key` files into a single `.pfx` — Windows' built-in `certutil -mergepfx`,
   or `openssl`.
2. Set `client_certificate_file` to the merged `.pfx` plus `client_certificate_password` on the web
   connection.
3. Do the OAuth client-credentials token exchange through a web connection endpoint (or
   `authentication_type = oauth` with an `oauth_server_id`, once the field gap above is resolved) — a
   Web Connection supports certificates end-to-end where the dedicated OAuth connector does not.
