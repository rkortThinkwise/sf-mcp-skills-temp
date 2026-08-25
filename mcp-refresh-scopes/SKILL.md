---
name: mcp-refresh-scopes
description: Reload OAuth scopes for an HTTP MCP connector (e.g. sf_mcp) by querying its /mcp endpoint's OAuth protected-resource metadata and syncing the local "scopes" config to match what the server currently advertises. Use when a connector's available scopes may have changed, when tools are unexpectedly missing/denied, or when the user asks to "refresh"/"reload"/"resync" MCP scopes.
---

# MCP scope refresh

Syncs the locally configured OAuth `scopes` string for an HTTP-type MCP server entry to whatever
the server itself currently advertises, so the connector's authorized scope list never silently
drifts out of date with the server.

## Steps

1. **Locate the connector config.** MCP server entries with `oauth` blocks live under
   `mcpServers` in `~/.claude.json`, nested at `projects["<project path>"].mcpServers.<name>`
   (project-scoped) — there may also be a user-level `mcpServers` key at the top of `.claude.json`,
   or a `.mcp.json` in the repo (project-shared scope). Find the entry by name (e.g. `sf_mcp`). If
   the user didn't name a connector, list all `mcpServers` entries that have an `oauth` block and
   ask which one (skip this ask if there's exactly one).

   Note the entry's `url` (e.g. `https://host/base_path/mcp`) and current `oauth.scopes` string.

2. **Query the server for its current scopes.** Derive the resource base by stripping the trailing
   `/mcp` path segment from `url`, then GET the OAuth protected-resource metadata document (RFC
   9728):

   ```
   curl -sS "<base>/.well-known/oauth-protected-resource"
   ```

   This endpoint is unauthenticated and returns JSON with a `scopes_supported` array — this is
   "the /mcp endpoint's scopes." (If that 404s, try appending the MCP path itself:
   `<base>/.well-known/oauth-protected-resource/mcp`.)

3. **Diff against the local config.** Compare the server's `scopes_supported` array against the
   current space-separated `oauth.scopes` string for that entry.

   - If they already match, say so and stop — nothing to change. **A "no drift" result here points at
     a different authorization layer** — internal Software Factory role/entity rights (row/table-level
     grants inside a domain the OAuth scope already covers) are a separate gate underneath the scope,
     and this skill can't diagnose or fix them; a write that still fails after a clean scope match is a
     Software Factory permissions question, not a connector config problem.
   - If the server list is a **strict superset** (adding scopes) or removes scopes the connector
     currently has, treat this as a real authorization change, not a mechanical sync.

4. **Confirm before widening access.** If new scopes would be added — especially anything that
   reads like a write/approve/send/manage/create/delete capability — use AskUserQuestion to
   show what's being added/removed and confirm before touching the file. Don't silently expand
   an OAuth grant. If scopes are only being removed/narrowed, a lighter confirmation is fine.

5. **Update the config file precisely.** Edit `oauth.scopes` in place for that one connector entry
   using a targeted string replacement (Read the exact surrounding lines first, then Edit) — do
   not round-trip the whole `.claude.json` through a JSON parser/serializer, since it's a large
   shared file and reserializing it risks reformatting unrelated content. Join the new scope list
   with single spaces, matching the server's ordering.

6. **Validate.** After editing, parse the file (e.g.
   `Get-Content -Raw path | ConvertFrom-Json`) to confirm it's still valid JSON and that the
   updated `oauth.scopes` value reads back correctly.

7. **Tell the user what changed and what's next.** Report old vs. new scope count/list briefly.
   Remind them that this only updates the stored config — any existing OAuth token was issued
   under the old scope, so the connector likely needs to reconnect/re-authenticate (e.g. via
   `/mcp` reconnect in Claude Code) before the new scopes actually take effect.
