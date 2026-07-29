---
name: perstat-status
description: >
  Report the current state of services monitored by Perstat. Use when the user
  asks "is anything down", "what's our status", "how is production doing", "check
  our monitors", "are there any open incidents", "what happened overnight", "is
  the site up", or wants a health check before or after a deploy. Read-only:
  reports what is happening without changing anything.
metadata:
  version: "0.1.0"
---

Read-only. Never acknowledge or resolve an incident from this skill; that is the
`handle-incident` skill, and it requires explicit confirmation.

## Start wide, then narrow

Call `get_organization_summary` first. One call gives monitor counts by state,
open incident count, and a `headline` string. That is usually the whole answer to
"is anything broken".

Then drill in only as far as the question needs:

- Something is down or degraded: `list_monitors` with `status: "down"` or
  `status: "degraded"` to name them
- Incidents are open: `list_incidents` (defaults to open, newest first), then
  `get_incident` for detail on the ones that matter
- Asked about one service: `list_monitors` to find its `mon_…`, then
  `get_monitor` for config, regions and recent incidents
- Asked how projects are laid out, or which project something sits in:
  `list_projects`, which returns each project with its monitor count
- Asked what customers can see: `list_status_pages`

If the key is narrowed to specific projects, `get_organization_summary` will not
be available. Fall back to `list_monitors` and `list_incidents`, and say that the
view covers only the key's projects, so "everything is fine" is not a claim about
the whole organization.

## Read the counts honestly

List tools return `total_matching` and `truncated` alongside the rows. When
`truncated` is true, the rows are a sample, not the set. Report the real total and
say the list was cut, rather than counting the rows you received.

Distinguish the states, because they mean different things:

- `down`: confirmed failure
- `degraded`: working but impaired, for example a missing security header or a
  certificate close to expiry
- `unknown`: no confirmed result yet. A monitor that has never reported, such as
  a heartbeat nobody pings, sits here. This is not the same as healthy, and it is
  worth calling out rather than folding into a green summary
- `disabled`: paused, not being checked at all. Also not healthy, just quiet

Archived monitors are not in the normal listings or the organization summary, by
design, because they represent retired services. They are not lost though:
`list_monitors` with `archived: true` is the archive view, and `get_monitor`
resolves an archived monitor by id. If a user asks about a service you cannot
find, check the archive before saying it does not exist.

## Reporting

Lead with the answer. "Everything is up, no open incidents" needs no table.

When something is wrong, give what a person needs to act: which service, what
state, since when, whether an incident is open, and whether anyone has taken it.
An unacknowledged open incident is the thing worth surfacing first, because it
means nobody has picked it up yet.

Note paused and unknown monitors when reporting overall health. An organization
where half the monitors are paused is not the same as one where everything is
green, and a summary that hides the difference is misleading.
