---
name: handle-incident
description: >
  Acknowledge or resolve an open Perstat incident. Use when the user says
  "acknowledge that incident", "ack inc_…", "I'm on it", "take this incident",
  "resolve the incident", "close that incident", "mark it fixed", or "silence the
  alert for X". Both actions are visible to the on-call team and change alerting,
  status pages and availability figures, so both require explicit confirmation
  first.
metadata:
  version: "0.1.0"
---

These two tools reach real people. Acknowledging stops an escalation that might
otherwise wake up the next person in the rotation. Resolving closes an incident
that customers may be watching on a status page, and feeds availability numbers.
Neither is reversible from here.

## Confirm before acting, every time

Do not call `acknowledge_incident` or `resolve_incident` on inference. Confirm
explicitly, even when the user's phrasing sounds like an instruction, unless they
have already named the specific incident and the specific action in this
conversation.

Before asking, look: `list_incidents` to find it, `get_incident` for detail. Then
state what you are about to do and what follows from it:

> `inc_…`, "api.example.com health" has been down for 22 minutes, currently
> unacknowledged. Acknowledging stops the alerting and escalation and puts your
> name on it for the team. Go ahead?

Never acknowledge or resolve incidents in bulk from a vague instruction like
"clean up the incident list". Ask which ones.

## Acknowledging

`acknowledge_incident` with the `inc_…` means "a human has taken this". It stops
alerting and escalation. It does not fix anything and it does not close the
incident.

The action is attributed to the person who created the API key, not to the agent,
and the incident timeline records that it came via MCP. Worth telling the user
plainly: their teammates will see their name on this. If they are acknowledging
on someone else's behalf, that is not what will show up.

## Resolving

`resolve_incident` closes an open incident by hand. Use it when the underlying
problem is genuinely fixed, or when the incident was noise.

Before resolving, check whether the monitor is actually healthy again. Call
`get_monitor` and look at its current state. Resolving an incident while the
monitor is still `down` usually just means a new incident opens shortly after,
and in the meantime the status page has told customers everything is fine. If the
monitor is still failing, say so and ask whether to resolve anyway; there are
legitimate reasons, such as a monitor misconfiguration being the actual fault.

## When the action fails

A write can fail even though reads work. Scopes govern reading; writes
additionally re-check the human behind the key, who must still be a member of the
organization with responder rights or higher for incident actions. If they were
downgraded, removed, or their account was deleted, incident writes stop
immediately without the key being revoked.

Report that plainly rather than retrying. It is an access change, and the fix is a
conversation with an Owner or Admin, not another call.

## After acting

Say what was done and what state the incident is now in. If an acknowledgement
stopped an escalation, say that, because it tells the user nobody else will be
paged. If work remains, say what: an acknowledged incident is claimed, not fixed.
