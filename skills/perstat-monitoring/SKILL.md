---
name: perstat-monitoring
description: >
  This skill should be used whenever Perstat monitoring comes up: when the user says
  "Perstat", asks about "monitors", "uptime checks", "health checks", "heartbeat",
  "dead man's switch", "SSL certificate expiry", "DNS hygiene", "security headers",
  "check regions", or asks what a service ought to have monitored. Provides the
  monitor type catalogue with the exact config shape per type, check regions,
  intervals, ID formats, and the rules the Perstat MCP server enforces. Read this
  before calling any perstat MCP tool, because the server's create_monitor tool
  deliberately ships no schema for its config field.
metadata:
  version: "0.1.0"
---

# Perstat

Perstat watches services from six continents and turns confirmed failures into
incidents, alerts and status pages. This plugin talks to it over MCP, so monitors
can be created and managed from inside the project they belong to.

Consult this skill before any `perstat` tool call. The server's `create_monitor`
schema leaves `config` free-form on purpose, since the thirteen types differ too
much for one schema. The type catalogue below fills that gap.

## Connecting

The MCP server lives at `https://api.perstat.io/mcp` and authenticates with an
organization API key only:

```
Authorization: Bearer pst_…
```

A browser session does not count. A key belongs to exactly one organization and
carries its own scopes, which keeps an agent's rights predictable.

Set `PERSTAT_API_KEY` in the environment. Keys are created by an Owner or Admin
under `/o/<org>/api-keys`. If tool calls fail with an auth error, the key is
missing, revoked, or scoped too narrowly. See `references/setup.md`.

## Identifiers

| Prefix | Thing | Where it comes from |
| --- | --- | --- |
| `prj_…` | Project | `list_projects` |
| `mon_…` | Monitor | `list_monitors`, `get_monitor` |
| `inc_…` | Incident | `list_incidents` |

**Finding a project ID.** Call `list_projects`. It takes no arguments and returns
every project with its monitor count. It works in an organization that has no
monitors yet, which is exactly when `create_monitor` needs it. Never invent or
guess an ID.

## Tools and what they need

| Tool | Scope | Notes |
| --- | --- | --- |
| `get_organization_summary` | `monitors:read` | Best first call. Org-wide keys only |
| `list_projects` | `monitors:read` | Projects with monitor counts. No arguments. The way to a `prj_…` |
| `list_monitors` | `monitors:read` | Filter by `project_id`, `status`, `limit`. `archived: true` switches to the archive view |
| `get_monitor` | `monitors:read` | Full config, regions, recent incidents. Resolves archived monitors too |
| `list_incidents` | `incidents:read` | Defaults to open, newest first |
| `get_incident` | `incidents:read` | One incident in detail |
| `acknowledge_incident` | `incidents:write` | Stops alerting and escalation |
| `resolve_incident` | `incidents:write` | Closes an open incident by hand |
| `list_status_pages` | `status-pages:read` | Org-wide keys only |
| `create_monitor` | `monitors:write` | Org-wide keys only. Not idempotent |
| `update_monitor` | `monitors:write` | Replaces `config` wholesale |
| `set_monitor_enabled` | `monitors:write` | Pause or resume. Does not free a plan slot |
| `archive_monitor` | `monitors:write` | Retire a monitor. Frees its plan slot |
| `restore_monitor` | `monitors:write` | Bring one back. It returns paused |

`write` implies `read` of the same resource, never across resources.

## Rules that will bite if ignored

**Nothing is deleted, but monitors can be archived.** There is no hard delete
here for a concrete reason: removing a monitor cascades through eleven tables and
takes the incident history, the SLA daily rollups and the exclusion windows with
it. In a product whose whole point is evidence, an agent should not be able to do
that. `archive_monitor` grants the same wish without the damage.

Three states, and the difference matters:

| State | Checks | In listings | Plan slot | History |
| --- | --- | --- | --- | --- |
| Active | yes | yes | taken | kept |
| Paused (`set_monitor_enabled`, `enabled: false`) | no | yes | **still taken** | kept |
| Archived (`archive_monitor`) | no | no | **freed** | kept |

Pausing is for "quiet for now, we still mean it", such as an environment being
rebuilt. Archiving is for "this service is gone", such as a decommission. Only
archiving frees quota, so a user who is at their plan limit and pausing monitors
to make room is doing the wrong thing; tell them.

Archiving also releases the name, so a new monitor can take it. `restore_monitor`
brings an archived monitor back, and it **returns paused**: checking resumes only
after an explicit `set_monitor_enabled`.

**Both pausing and archiving close the monitor's open incidents**, and on-call
escalation for it stops with them. Nobody gets paged for a service that was
retired or quieted on purpose. This is worth stating when reporting the action,
because it is the part a user actually cares about at 3am.

**Finding archived monitors.** `list_monitors` with `archived: true` shows them,
and that is the way to an id for `restore_monitor`. It is a separate view, not an
additive filter: `archived: true` lists archived monitors *instead of* active
ones, so it cannot be combined to get both at once. Rows in that view carry no
project or status, since neither applies to a retired service. `get_monitor`
resolves an archived monitor as well, so its config and history can be inspected
before deciding to restore.

**Writing to an archived monitor is refused, and says so.** `update_monitor`,
`set_monitor_enabled` and a second `archive_monitor` all answer with `Monitor …
is archived. Call restore_monitor first, then change it.` That is deliberately
not a "not found" error, because the monitor does exist, it is just out of
service. Restore first, then make the change.

Real deletion happens in the web app, where a human sees the consequences.

**`create_monitor` is not idempotent.** A second identical call creates a second
monitor or fails on the name. Always `list_monitors` first to check whether the
monitor already exists.

**`update_monitor` replaces `config` entirely, it does not merge.** Call
`get_monitor` first, take its `config`, change the one field, and send the whole
object back. Sending only the changed key silently drops every other setting.

**Narrowed keys see less.** A key restricted to certain projects or monitors sees
only its own resources, and unknown IDs come back as "no monitor with that ID"
rather than "forbidden". Org-wide views and `create_monitor` are not offered to
such keys at all. If `create_monitor` is missing from the tool list, the key is
narrowed; the user needs an org-wide key.

**Writes re-check the human.** Every write verifies that the person who created
the key is still a member with the needed role. Managing monitors needs Owner,
Admin or Developer. Incident actions need responder rights or higher. The action
is attributed to that person in the activity log, with a note that it came via
MCP, so writes are never anonymous.

**Out of scope for MCP entirely:** status page editing, connectors, API keys,
members, plan. Agents stay on monitoring and incidents.

## Monitor types

Thirteen types. Full config shapes with examples are in
`references/monitor-types.md`. Read that file before building any `config`.

| Type | Watches | Key config |
| --- | --- | --- |
| `http` | An HTTP endpoint, plus optional TLS, header and DNS sub-checks | `url` |
| `http_headers` | Security headers on a URL | `url`, `headers` |
| `tcp` | A TCP port accepting connections | `host`, `port` |
| `dns` | A DNS record resolving to what it should | `name`, `record_type` |
| `dns_hygiene` | SPF, DMARC and CAA of a domain | `domain` |
| `domain` | Zone health, registration expiry, DNSSEC | `domain` |
| `ssl_cert` | Certificate validity and expiry | `host`, `port` |
| `smtp` | A mail server answering | `host`, `port` |
| `imap` | A mailbox server answering | `host`, `port` |
| `ping` | ICMP reachability | `host` |
| `traceroute` | Path to a host | `host` |
| `agent` | A machine's availability, CPU, memory, disk or a service | `agent_id`, `metric` |
| `heartbeat` | A job that must check in, a dead man's switch | `period_seconds` |

`agent` and `heartbeat` are evaluated server-side and take no probe regions.

## Regions and intervals

Regions are continents: `na`, `eu`, `as`, `sa`, `af`, `oce`. Omitting `regions`
lets the plan pick a sensible default, which is usually the right move. A monitor
needs a quorum of regions to confirm a failure, so a single region means a single
point of view.

`interval_seconds` defaults to 300. Common values are 30, 60, 300, 900 and 3600.
The plan raises values that are too small for it, so a rejected interval is a
plan limit, not a bad request.

## Additional resources

- `references/monitor-types.md`: config shape and a worked example for all
  thirteen types, plus the shared sub-checks (TLS, security headers, DNS hygiene,
  address families).
- `references/setup.md`: getting an API key, choosing scopes, connecting the
  server, and what each failure mode means.
