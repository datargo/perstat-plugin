# Setup and troubleshooting

## Getting an API key

1. Sign in to Perstat as an Owner or Admin.
2. Go to `/o/<your-org>/api-keys`.
3. Create a key. It is shown once, at creation time, and never again.
4. Choose scopes (see below).
5. Put it in the environment as `PERSTAT_API_KEY`.

The key looks like `pst_…`. A browser session cannot be used instead: a key
belongs to exactly one organization, which removes the ambiguity of a person who
is a member of several, and it carries scopes, which makes an agent's rights
predictable.

## Choosing scopes

Grant the least the work needs.

| Scope | Unlocks |
| --- | --- |
| `monitors:read` | `get_organization_summary`, `list_projects`, `list_monitors`, `get_monitor` |
| `monitors:write` | `create_monitor`, `update_monitor`, `set_monitor_enabled`, `archive_monitor`, `restore_monitor` |
| `incidents:read` | `list_incidents`, `get_incident` |
| `incidents:write` | `acknowledge_incident`, `resolve_incident` |
| `status-pages:read` | `list_status_pages` |

`write` implies `read` of the same resource, never across resources. A key with
`monitors:write` still needs `incidents:read` to see incidents.

For a key that only creates and maintains monitors from a project repository,
`monitors:write` plus `incidents:read` is a good default: enough to manage
coverage and to see the consequences, not enough to close an incident nobody
looked at.

## Org-wide versus narrowed keys

A key can be narrowed to specific projects or monitors. A narrowed key:

- sees only its own resources in `list_monitors` and `list_incidents`
- gets "no monitor with that ID" for anything outside its reach, rather than a
  permission error, so it cannot be used to probe for what exists
- is not offered `get_organization_summary`, `list_status_pages` or
  `create_monitor` at all. `list_projects` it does get, filtered to its own
  projects

Creating monitors needs an org-wide key. A narrowed key would otherwise write
into projects it knows nothing about.

## Connecting without this plugin

The plugin ships the server config. To connect Claude Code by hand instead:

```bash
claude mcp add --transport http perstat https://api.perstat.io/mcp --header "Authorization: Bearer pst_…"
```

## What failures mean

**The `perstat` tools are missing entirely.** The MCP server is not connected.
Check that `PERSTAT_API_KEY` is set in the environment the client was started
from, then restart the client.

**Every call returns an auth error.** The key is wrong, revoked, or belongs to a
different organization. Keys are shown once at creation; if it was not saved, a
new one has to be created.

**`create_monitor` is not in the tool list, other tools are.** The key is
narrowed to projects or monitors, or lacks `monitors:write`. An org-wide key with
`monitors:write` is required.

**A write fails but reads work.** Scopes govern reading; writing additionally
re-checks the human who created the key. That person must still be a member of
the organization with a sufficient role: Owner, Admin or Developer for monitors,
responder or higher for incidents. If they were downgraded, removed, or their
account was deleted, writes stop immediately without anyone revoking the key.

**A list result looks short.** List tools return `total_matching` and `truncated`
alongside the rows. Check them before reporting a count. A truncated answer is
marked as such rather than pretending to be complete.

**A tool returns `isError: true` with prose.** That is a tool-level failure, not
a protocol error, and the text explains what went wrong. Read it and adjust.
Protocol errors (`-32601`, `-32602`, `-32700`) mean a genuinely malformed call.

**Batched JSON-RPC requests are rejected.** They were removed from the protocol
in revision `2025-06-18`. Send one message per request.

## Attribution

Writes are attributed to the person who created the key, with the origin recorded
as MCP. Acknowledging an incident through an agent therefore shows a name in the
incident timeline, not just a token. This is worth telling users: their teammates
will see their name on actions an agent took with their key.
