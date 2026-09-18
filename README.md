# Perstat for Claude

Create and manage monitoring for your services without leaving the project they
belong to. Ask Claude what should be watched, have it read your repository and
propose monitors, then create them. Check status before a deploy, and take an
incident when one opens.

[Perstat](https://perstat.io) checks services from six continents, confirms
failures across regions before alerting, and drives incidents, on-call escalation
and public status pages.

## Install

You need a [Perstat](https://perstat.io) account and Claude Code.

**1. Add the marketplace and install the plugin:**

```bash
claude plugin marketplace add datargo/perstat-plugin
```

```bash
claude plugin install perstat@datargo
```

Or do the same interactively with `/plugin` inside Claude Code.

**2. Sign in when Claude asks.** That is the whole setup. On the first
`perstat` tool call the server answers `401`, Claude opens Datargo ID in your
browser, you pick the organization the connection should apply to, and you are
back. No key to copy, nothing to paste, nothing in your shell profile.

The connection is bound to that one organization and to you: it can never do
more than your own role allows. If someone removes you from the organization,
the connection stops working immediately, without anyone having to revoke a
token.

**3. Check that it took.** Run `claude mcp list` and look for `perstat`
reporting `Connected`.

You can see and end connections any time at
<https://id.datargo.com/connections>.

### Running without a browser (CI, cron, servers)

The sign-in flow needs a human. Where there is none, use an API key instead:
create one in Perstat under `/o/<your-org>/api-keys` (Owner or Admin, shown
once) and hand it to the server as a header:

```bash
claude mcp add --transport http perstat https://api.perstat.io/mcp \
  --header "Authorization: Bearer pst_your_key_here"
```

A key is also the better fit when you want to narrow access to single projects
or monitors, which the browser flow does not offer. Do not put it in a project
`.env` file: Claude Code does not read one, and a key inside a repository is a
key waiting to be committed.

That is all. The plugin brings the server configuration with it, so there is no
MCP setup to do by hand. To update later, run `claude plugin update perstat@datargo`.

## What it does

**Sets up monitoring from what your project actually exposes.** Ask "set up
monitoring for this service" and Claude reads your deploy configs, manifests,
CI workflows and cron entries to find the URLs, domains, ports, certificates and
scheduled jobs worth watching. It proposes a set, waits for your yes, then
creates them.

**Knows all thirteen monitor types.** HTTP endpoints, TCP ports, DNS records,
domain and zone health, TLS certificates, SMTP and IMAP, ping and traceroute,
security header posture, SPF/DMARC/CAA hygiene, host agents, and heartbeats for
jobs that have nothing to connect to. The MCP server deliberately ships no schema
for the per-type settings, so this plugin carries them.

**Maintains them safely.** Change an interval, add a region, pause a noisy
monitor, archive one whose service you decommissioned. Editing goes through
read-then-write, because the API replaces a monitor's settings wholesale rather
than merging.

**Reports status.** "Is anything down?", "how did we do overnight?", "check
production before I deploy".

**Handles incidents.** Acknowledge to stop an escalation, resolve when it is
fixed. Both ask you first, because both are visible to your on-call team.

## Using it from CLAUDE.md

Put standing instructions in your project's `CLAUDE.md` and Claude will follow
them without being asked each time.

Keep coverage current as the project changes:

```markdown
## Monitoring

This project's services are monitored in Perstat, project `prj_…`.
When a new public endpoint, domain or scheduled job is added, propose a
matching Perstat monitor before considering the change complete.
When a service is decommissioned, archive its monitor rather than
leaving it paused, so it stops using a slot in our plan.
```

Check before shipping:

```markdown
## Before deploying

Check Perstat status first. Do not deploy while an incident is open
on a service this change touches.
```

House rules for how monitors get made:

```markdown
## Monitoring conventions

- Name monitors "<host> <what>", for example "api.example.com health".
- Prefer one HTTP monitor with TLS and header sub-checks over several
  monitors on the same URL.
- Every cron job and queue worker gets a heartbeat monitor.
- Missing security headers are `degraded`, never `down`.
```

You can also just ask, with no setup:

- "What should we be monitoring that we aren't?"
- "Add a heartbeat for the nightly backup, it runs at 02:00"
- "Our cert expires soon, are we watching it?"
- "Pause the staging monitors, we're rebuilding that environment"
- "We shut down the legacy API, archive its monitors"
- "Is anything down right now?"

## Skills

| Skill | Does |
| --- | --- |
| `perstat-monitoring` | Reference knowledge: monitor types and their settings, regions, intervals, identifiers, the rules the API enforces. Loads automatically when Perstat comes up |
| `manage-monitors` | Creates monitors from what a repository exposes, edits them safely, pauses, archives and restores |
| `perstat-status` | Reports current state of services and open incidents. Read-only |
| `handle-incident` | Acknowledges and resolves incidents, with confirmation |

## Things worth knowing

**Nothing is deleted, but monitors can be archived.** Perstat has no hard delete
here on purpose: removing a monitor would cascade through eleven tables and take
its incident history, SLA rollups and exclusion windows with it, which defeats
the point of a product built on evidence. Archiving retires a service instead. It
stops being checked, leaves the listings, frees its slot in your plan, and keeps
everything it ever recorded. Ask for "what did we archive?" to see the archive,
and it can be restored from there.

**Pausing and archiving are not the same thing.** A paused monitor still occupies
a slot in your plan; an archived one does not. If you are at your plan limit,
archiving is what makes room. Restoring brings a monitor back paused, so checking
resumes only when you say so.

**Creating a monitor is not idempotent.** Claude checks for duplicates before
creating, but two identical requests in a row will produce two monitors.

**Your name goes on writes.** Actions taken with your key are attributed in
Perstat's activity log to you, the person who created it, with a note that they
came via MCP. Your teammates will see your name on an incident an agent
acknowledged.

**Access is checked twice.** The key's scopes govern what can be read. Writing
additionally re-checks that you are still a member of the organization with a
sufficient role. Lose the role and writes stop immediately, without the key
needing to be revoked.

**Heartbeat monitors need wiring.** `create_monitor` returns the ping URL of a new
heartbeat, and `get_heartbeat_endpoint` returns it again later; call it from the
job. Until the first ping arrives the monitor reads `unknown`, which looks like
coverage but is not. The URL is a credential: keep it out of chat and repositories,
and rotate it in the web app if the conversation that saw it is not trusted.

**Status pages, connectors, API keys, members and billing are not reachable**
from here, by design. Agents stay on monitoring and incidents.

## Troubleshooting

**The `perstat` tools are missing entirely.** MCP servers are wired up when a
session starts, so a plugin installed mid-session only appears after you start a
new one. Check with `claude mcp list`.

**A tool call fails with `401`.** That is the sign-in prompt, not a fault. Follow
the browser flow. If no browser opens, you are probably running headless; use an
API key instead (see above).

**A tool call fails with `403` and says you have no Perstat account.** The
Datargo ID you signed in with has never been used on the web. Sign in once at
<https://app.perstat.io>, then reconnect.

**A tool call fails with `403` and mentions membership.** Someone removed you
from the organization the connection was made for. This is the intended kill
switch. Reconnect and pick an organization you are still in.

**`create_monitor` is missing while other tools work.** You are using an API key
that is narrowed to specific projects, or that lacks `monitors:write`. The
browser flow never narrows this way; an org-wide key does not either.

More in `skills/perstat-monitoring/references/setup.md`.
