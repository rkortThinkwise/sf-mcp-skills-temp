---
name: thinkwise-install-mcp
description: Add and authenticate a custom HTTP MCP connector in Claude Code end-to-end, including OAuth discovery and scope pinning, asking the user step by step rather than assuming any config value. Use when asked to add/install/set up/register a new MCP connector or server in Claude Code, especially an Indicium/sf_mcp-style OAuth-protected HTTP MCP endpoint.
---

install an mcp connector using the following instructions:

# Install an MCP connector in Claude Code

Set up and authenticate a custom MCP connector in Claude Code. Ask the
questions in Step 0 one at a time before doing anything — do not assume
defaults for values the user hasn't given you, except where marked "default."

## Step 0 — Gather configuration

Ask, in order, waiting for each answer before asking the next:

1. **Server name** — the alias to register this connector under in Claude
   Code (e.g. `sf_mcp`). This name must be used consistently in every step
   below (add, get, config lookup, `/mcp` selection) — do not let the add
   command use a different name than later steps reference.
2. **MCP endpoint URL** — the full transport URL (e.g.
   `https://<host>/<path>/mcp`).
3. **OAuth client ID.**
4. **OAuth client secret** — tell the user it will only be exported into the
   current shell session as an env var, never echoed, logged, or written to
   a file.
5. **Callback port** (default: `8080`) — only needs to change if the
   server's OAuth client has a different redirect URI pre-registered.
6. **OAuth discovery document URL** (default:
   `<mcp base URL without /mcp>/.well-known/openid-configuration`) — ask
   only if the user says the default path doesn't apply.
7. **Config scope** (default: `user`) — `user` registers the connector once
   for every project on this machine; `local` restricts it to the current
   project only. Ask only if the user wants something other than `user`.

Call the collected values `{{SERVER_NAME}}`, `{{MCP_URL}}`, `{{CLIENT_ID}}`,
`{{CLIENT_SECRET}}`, `{{CALLBACK_PORT}}`, `{{DISCOVERY_URL}}`, `{{SCOPE}}`
below. Derive `{{RESOURCE_BASE}}` by stripping the trailing `/mcp` path
segment from `{{MCP_URL}}`.

## Step 1 — Export the client secret for this shell session

The variable name is `MCP_CLIENT_SECRET` — fixed, not derived from the
server name. `claude mcp add --client-secret` reads only this name; anything
else fails with *"No TTY available to prompt for client secret. Set
MCP_CLIENT_SECRET env var instead."*

(bash)
```
export MCP_CLIENT_SECRET="{{CLIENT_SECRET}}"
```
(PowerShell)
```
$env:MCP_CLIENT_SECRET = "{{CLIENT_SECRET}}"
```

Agent shells generally do not persist environment variables between tool
calls. Run this export and the Step 2 `add` in a **single** invocation
(`;`-joined in PowerShell, `&&`-joined in bash) rather than as two steps.

## Step 2 — Add the MCP server

```
claude mcp add -s {{SCOPE}} --transport http {{SERVER_NAME}} {{MCP_URL}} \
  --client-id {{CLIENT_ID}} \
  --client-secret \
  --callback-port {{CALLBACK_PORT}}
```
`-s {{SCOPE}}` is not optional — with no `-s`, `claude mcp add` defaults to
`local`, which registers the server for the current project only.
`--client-secret` with no value reads from the env var set in Step 1.
`--callback-port` pins the OAuth redirect URI to
`http://localhost:{{CALLBACK_PORT}}/callback` — required whenever the
server's OAuth client has that exact redirect URI pre-registered.

## Step 3 — Add discovery metadata

The entry's location in `~/.claude.json` depends on the scope used in
Step 2:

| Scope | Path in `~/.claude.json` |
|---|---|
| `user`  | top-level `"mcpServers"` → `"{{SERVER_NAME}}"` → `"oauth"` |
| `local` | `"projects"` → `"<project path>"` → `"mcpServers"` → `"{{SERVER_NAME}}"` → `"oauth"` |

Locate the entry by searching the file for `"{{SERVER_NAME}}"` rather than
assuming either path. Under its `"oauth"` object, add:
```
"authServerMetadataUrl": "{{DISCOVERY_URL}}"
```

## Step 4 — Discover and pin scopes dynamically (do not hardcode)

Do not type out a fixed scope list. Query the server itself for what it
currently advertises, then write that into the config:

1. **Query the server for its current scopes.** GET the OAuth
   protected-resource metadata document (RFC 9728), which is unauthenticated:
   ```
   curl -sS "{{RESOURCE_BASE}}/.well-known/oauth-protected-resource"
   ```
   This returns JSON with a `scopes_supported` array. If it 404s, retry
   against the MCP path itself:
   ```
   curl -sS "{{RESOURCE_BASE}}/.well-known/oauth-protected-resource/mcp"
   ```
   If both fail, ask the user to paste the scope list manually and skip to
   step 5 below.

2. **Check for an existing scopes value.** Read `oauth.scopes` at the same
   scope-determined location as Step 3 — top-level
   `mcpServers.{{SERVER_NAME}}.oauth.scopes` for `user` scope,
   `projects["<project path>"].mcpServers.{{SERVER_NAME}}.oauth.scopes` for
   `local` — or a `.mcp.json` in the repo for project scope. It's likely
   empty/absent on a fresh add — that's expected, just treat the server's
   list as entirely new.

3. **Diff before writing.** Compare the server's `scopes_supported` array
   against any existing `oauth.scopes` string.
   - If they already match, say so and skip to Step 5 — nothing to change.
   - If this is a fresh add, or the server list only adds scopes, note what's
     being granted.
   - If scopes are being removed/narrowed from a prior value, note that too.

4. **Confirm before granting.** Since this is a first-time authorization
   (or a widening one), use AskUserQuestion to show the discovered scope
   list and confirm before writing it — especially anything that reads like
   a write/approve/send/manage/create/delete capability. Don't silently
   apply an OAuth grant the user hasn't seen.

5. **Write the config precisely.** Set `oauth.scopes` for
   `{{SERVER_NAME}}` to the confirmed list, joined with single spaces, in
   the server's own ordering:
   ```
   "scopes": "<scope1> <scope2> <scope3> ..."
   ```
   Do this with a targeted string replacement (Read the exact surrounding
   lines first, then Edit) — do not round-trip the whole `~/.claude.json`
   through a JSON parser/serializer, since it's a large shared file and
   reserializing risks reformatting unrelated content.

6. **Validate.** After editing, parse the file (e.g.
   `Get-Content -Raw ~/.claude.json | ConvertFrom-Json` or
   `python -c "import json; json.load(open(...))"`) to confirm it's still
   valid JSON and that `oauth.scopes` reads back as expected.

## Step 5 — Verify registration

```
claude mcp get {{SERVER_NAME}}
```
Expect `Status: Needs authentication`, the configured `client_id`,
`callback_port {{CALLBACK_PORT}}`, and a `Scope:` line matching
`{{SCOPE}}` (`User config (available in all your projects)` vs
`Local config (private to you in this project)`).

If `{{SERVER_NAME}}` is missing entirely, first confirm which scope it was
actually added under — re-run `claude mcp get {{SERVER_NAME}}` and read the
`Scope:` line, or `claude mcp list` from the directory in question. Do
**not** resolve a lookup miss by duplicating the entry under several
project-path key variants: `JSON.parse` keeps only the *last* duplicate
key, so the added entries are inert, and strict parsers (PowerShell's
`ConvertFrom-Json`) reject the file outright. If a project-scoped entry is
genuinely unreachable, re-add it with `-s user`.

## Step 6 — Authenticate

Claude Code enumerates MCP servers at session start, so a server added
mid-session will not appear in the current one regardless of what the config
says. Tell the user to **restart or reload the session first**, then run
`/mcp`, select `{{SERVER_NAME}}`, and complete the OAuth login in the
browser window that opens. Claude Code should then report the server as
connected.

Remind the user: this only registers the scopes in config — the actual
OAuth token is issued during this authentication step, so it must run
*after* Step 4's scopes are written, not before.

## Troubleshooting

If authentication fails with "Client secret validation failed", **remove
and re-add** rather than editing in place:
```
claude mcp remove {{SERVER_NAME}} -s {{SCOPE}}
```
then repeat Step 2. Claude Code links the stored client secret to a hash of
the server's config — editing the URL or secret directly in
`~/.claude.json` without going through `claude mcp` can silently orphan
that link.

**Changing scope after the fact** requires a remove and re-add:
`claude mcp remove {{SERVER_NAME}} -s <old-scope>`, then repeat Step 2 with
the new `-s`. Steps 3 and 4 must then be **redone**, because `claude mcp
add` writes only `type`, `url`, `clientId`, and `callbackPort` — the
`authServerMetadataUrl` and `scopes` added by hand do not migrate to the
new config location and are silently lost.

**If the MCP URL 404s**, check the transport scheme before anything else. A
correctly configured OAuth-protected MCP endpoint answers an unauthenticated
POST with `401` and a `WWW-Authenticate` header; a plain `404` (especially
an IIS error page) usually means the path is not served over that scheme at
all. Retry the same URL over `https://` before assuming the path is wrong.
