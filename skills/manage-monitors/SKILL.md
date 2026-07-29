---
name: manage-monitors
description: >
  Set up and maintain Perstat monitors for a project. Use when the user asks to
  "set up monitoring", "monitor this service", "add a monitor", "what should we
  monitor here", "add a heartbeat for this cron job", "watch our certificate",
  "change the check interval", "pause that monitor", "resume monitoring", "stop
  monitoring staging", "archive that monitor", "we decommissioned that service",
  "bring back the archived monitor", "what did we archive", "show archived
  monitors", "we're at our plan limit", or "delete a monitor". Covers creating monitors from what a repository actually exposes,
  editing them safely, and choosing correctly between pausing and archiving.
metadata:
  version: "0.1.0"
---

Read the `perstat-monitoring` skill first, including
`references/monitor-types.md`, before building any `config`. The server ships no
schema for it.

## Always start by looking

Call `list_projects` for the `prj_…` that `create_monitor` needs. It takes no
arguments and returns every project with its monitor count, so it works on a
brand new organization too. If there is more than one project and the right one
is not obvious from the names, ask rather than picking.

Then call `list_monitors` to see what already exists. `create_monitor` is not
idempotent, so skipping this is how organizations end up with two monitors for
one URL.

## Creating monitors for a project

When the user asks to set up monitoring for a repository or service, work out
what it actually exposes rather than asking them to list it.

**Find the monitorable surfaces.** Look for:

- Public URLs and health endpoints, in READMEs, deploy configs, Kubernetes
  manifests, `docker-compose` files, CI workflows, Terraform, nginx or Caddy
  configs
- Custom domains and the certificates behind them
- Databases, caches, message brokers with a reachable port
- Mail sending, which implies SPF, DMARC and CAA are worth watching
- Scheduled work: cron entries, systemd timers, CI schedules, queue workers.
  These need `heartbeat` monitors, and they are the ones teams most often forget

**Propose before writing.** Present a table of what you intend to create: name,
type, target, interval, and one line on why. Then wait for confirmation. Creating
monitors costs plan quota and generates alerts that reach real people, so it is
not something to do silently.

Suggest fewer, better monitors. One `http` monitor with `tls_cert`,
`security_headers` and `dns_hygiene` sub-checks beats four separate monitors on
the same URL: when the host goes down it produces one incident instead of four.

**Naming.** Use names a person woken at 3am can parse: "api.example.com health",
not "http monitor 3". Names must be unique within the project.

**Then create them one at a time**, checking each result before the next. If one
fails, report which and why rather than continuing blindly.

**Regions and interval.** Omit `regions` unless the user has a reason, so the
plan picks its default. Omit `interval_seconds` to get 300 seconds. Do not send
`regions` for `agent` or `heartbeat` types, which are evaluated server-side.

**After creating a heartbeat monitor**, tell the user it stays `unknown` until
the job first pings it, and that the ping URL has to be fetched from the web app
and wired into the job. An unpinged heartbeat looks like coverage while providing
none.

## Editing an existing monitor

`update_monitor` replaces `config` wholesale. It does not merge.

The safe sequence, every time:

1. `get_monitor` for the current state
2. Take its `config` object entire
3. Change the one field
4. Send the whole object back

Sending just the changed key silently drops every other setting, and the loss is
invisible until the monitor stops catching what it used to.

`name`, `interval_seconds` and `regions` are independent fields: omitting them
leaves them unchanged, which is the normal behaviour and safe.

## Pausing, archiving, and the delete question

Two different tools for two different intentions. Choosing wrongly either wastes
plan quota or hides a service the team still cares about.

**Pause** by calling `set_monitor_enabled` with `enabled: false` when the service still matters and
should be quiet for a while: an environment being rebuilt, a noisy monitor
pending a fix, a planned migration. It stays in listings and **keeps occupying
its plan slot**.

**Archive** with `archive_monitor` when the service is gone: a decommission, a
retired endpoint, a project that ended. It leaves the listings, **frees the plan
slot**, and keeps its full history, availability figures and exclusion windows.
The name is released, so a new monitor may reuse it.

When a user is at their plan limit and wants room, archiving is the answer.
Pausing will not help and they will think it did.

**Either action closes the monitor's open incidents and stops its on-call
escalation.** Say so when reporting, especially if the monitor was down: the
useful sentence is "its open incident is closed and nobody will be paged for it",
not just "archived".

**Finding what was archived.** `list_monitors` with `archived: true` is the
archive view. Use it to get the `mon_…` for `restore_monitor`. It replaces the
active list rather than extending it, so to show both you need two calls.
`get_monitor` works on an archived monitor, which is worth doing before
restoring: check what it was watching and whether that still makes sense.

**Restoring.** `restore_monitor` brings an archived monitor back **paused**. Say
so, and offer the follow-up call to `set_monitor_enabled` with `enabled: true`, or the user will think
checking resumed when it did not.

**Changing an archived monitor does not work.** `update_monitor`,
`set_monitor_enabled` and a second `archive_monitor` all refuse with `Monitor …
is archived. Call restore_monitor first, then change it.` Read that as an
instruction, not a dead end: restore, change, then enable if wanted. Confirm with
the user before restoring something on their behalf, since it takes a plan slot
again.

Reporting the `mon_…` when you archive is still a courtesy worth doing, but it is
no longer the only way back.

**The delete question.** There is still no hard delete, and that is deliberate:
deleting cascades through eleven tables and takes the incident history, SLA
rollups and exclusion windows with it. When a user asks to delete, do not just
report the limitation. Ask what they mean. "We shut that service down" is
archiving, and that is almost always what they want. Genuine erasure happens in
the web app, where a human sees what goes with it.

## Before reporting success

Say what changed, in plain terms: which monitors now exist, what each watches,
and anything still needing a human, such as wiring a heartbeat ping or
installing an agent. If a monitor was created but cannot yet report, say so
rather than letting it read as finished coverage.
