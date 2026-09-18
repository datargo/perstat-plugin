# Monitor types and their config

The `config` object passed to `create_monitor` and `update_monitor` is
type-dependent. This file gives the shape for every type. Fields marked optional
can be left out entirely; do not send `null` for them.

When unsure about an existing monitor, call `get_monitor` and use its `config` as
the template. Remember that `update_monitor` replaces `config` wholesale, so
always start from the current one.

## Probe types

These run from the check regions.

### `http`

The workhorse. Watches an endpoint and can carry three optional sub-checks.

```json
{
  "url": "https://example.com/health",
  "expected_status": 200,
  "keyword": "ok",
  "expected_regex": "\"status\"\\s*:\\s*\"ok\"",
  "tls_cert": { "enabled": true, "warn_days": 21 },
  "security_headers": {
    "enabled": true,
    "headers": ["strict-transport-security", "content-security-policy"],
    "missing_severity": "degraded"
  },
  "dns_hygiene": { "enabled": true, "domain": "example.com" }
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `url` | yes | Full URL including scheme |
| `expected_status` | no | Exact status code. Omit to accept any 2xx |
| `keyword` | no | Body must contain this string |
| `expected_regex` | no | Body must match this regex |
| `tls_cert` | no | TLS sub-check, https URLs only. See below |
| `security_headers` | no | Header posture sub-check. See below |
| `dns_hygiene` | no | SPF, DMARC and CAA sub-check. `domain` defaults to the URL host |

Prefer one `http` monitor with sub-checks over three separate monitors for the
same URL. One monitor means one incident when the host goes down, not three.

### `http_headers`

Security header posture as a standalone monitor. Use when the endpoint's
availability is already covered elsewhere and only the headers matter.

```json
{
  "url": "https://example.com",
  "headers": ["strict-transport-security", "content-security-policy"],
  "missing_severity": "degraded"
}
```

At least one header is required. Valid keys, lowercase exactly as shown:

- `strict-transport-security` (HSTS)
- `content-security-policy` (CSP)
- `x-content-type-options`
- `x-frame-options`
- `referrer-policy`

`missing_severity` is `degraded` or `down`. Prefer `degraded`: a missing header
is a posture finding, not an outage, and paging someone at 3am for it trains them
to ignore the pager.

### `tcp`

```json
{ "host": "db.example.com", "port": 5432 }
```

Optional `tls_cert` sub-check. Optional `address_families`.

### `dns`

Checks that a record resolves to what it should.

```json
{
  "name": "example.com",
  "record_type": "A",
  "expected": "203.0.113.10",
  "expected_all": ["203.0.113.10", "203.0.113.11"],
  "expected_regex": "^203\\.0\\.113\\.",
  "cross_ref": { "monitor": "mon_…", "mode": "match" }
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | The name to resolve |
| `record_type` | yes | `A`, `AAAA`, `MX`, `TXT`, `CNAME` or `NS` |
| `expected` | no | One value that must be present |
| `expected_all` | no | The complete expected answer set |
| `expected_regex` | no | Answer must match |
| `cross_ref` | no | Compare against another monitor's resolved target |

Use `expected_all` when the full set matters, such as NS records. Use `expected`
when one value among several is enough.

### `dns_hygiene`

SPF, DMARC and CAA of a domain.

```json
{ "domain": "example.com" }
```

### `domain`

Zone and delegation health, registration expiry, DNSSEC.

```json
{
  "domain": "example.com",
  "expected_ns": ["ns1.example.net", "ns2.example.net"],
  "warn_days": 30
}
```

`warn_days` is how many days before registration expiry to start warning.

### `ssl_cert`

```json
{
  "host": "example.com",
  "port": 443,
  "warn_days": 21,
  "issuer_regex": "Let's Encrypt",
  "subject_regex": "example\\.com",
  "allow_self_signed": false
}
```

Only `host` is required. `port` defaults to the standard port for the type.

An `http` monitor on an https URL can carry `tls_cert` instead, which is usually
better. Reach for a standalone `ssl_cert` monitor when the host serves TLS but no
useful HTTP endpoint, or when the certificate deserves its own incident.

### `smtp` and `imap`

```json
{ "host": "mail.example.com", "port": 587, "tls_cert": { "enabled": true } }
```

`host` required, `port` optional, `tls_cert` optional.

### `ping`

```json
{ "host": "gateway.example.com" }
```

### `traceroute`

```json
{ "host": "gateway.example.com" }
```

Traceroute is a diagnostic view of the path, not a crisp up or down signal. Pair
it with `ping` or `http` rather than alerting on it alone.

## Server-side types

These are evaluated centrally. They take no probe regions, and the server sets
their interval. Do not send `regions` for them.

### `agent`

Binds to an installed Perstat host agent.

```json
{
  "agent_id": "…",
  "metric": "cpu",
  "threshold": 90,
  "breach_severity": "degraded",
  "notify_push": true
}
```

| `metric` | Extra fields | Meaning |
| --- | --- | --- |
| `availability` | `grace_seconds` (60 to 3600, default 180) | Host stopped checking in |
| `cpu` | `threshold` (0 to 100) | CPU percentage above threshold |
| `mem` | `threshold` (0 to 100) | Memory percentage above threshold |
| `disk` | `threshold` (0 to 100) | Disk percentage above threshold |
| `service` | `service_name`, `grace_seconds` (60 to 3600, default 120) | A named process stopped running |

`breach_severity` is `down` or `degraded`. `notify_push` toggles push
notification; in-app notification happens regardless.

`grace_seconds` exists so that deploy restarts do not fire incidents. Lowering it
below the service's restart time produces false alarms.

Requires an agent installed on the host, and `agent_id` has to come from an
existing agent. Ask the user rather than guessing.

### `heartbeat`

A dead man's switch. The job pings a secret URL; silence past
`period_seconds` plus `grace_seconds` marks it down. Right for cron jobs, batch
runs, backups and anything with no listening port.

```json
{
  "period_seconds": 3600,
  "grace_seconds": 600,
  "breach_severity": "down",
  "notify_push": true
}
```

`period_seconds` is how often the job is expected to check in, clamped server-side
to between 30 and 2592000 (30 days). `grace_seconds` is the tolerance on top.

Set `grace_seconds` to a real fraction of the period. A nightly backup with a
60 second grace will page on every slow night.

`create_monitor` returns the ping URL of a new heartbeat as `heartbeat.ping_url`
(plus `fail_url`, the same URL with `/fail`), and `get_heartbeat_endpoint`
returns it again later. Wire it into the job straight away; the monitor stays
`unknown` until the first ping arrives, and a heartbeat nobody pings is worse
than no monitor because it looks like coverage. The URL is a credential: whoever
holds it can report success and keep an outage green. Store it where the job
reads it, never in a repository or a log, and do not repeat it in chat.

## Shared sub-checks

### `tls_cert`

Available on `http` (https URLs), `tcp`, `smtp` and `imap`.

```json
{
  "enabled": true,
  "port": 443,
  "warn_days": 21,
  "issuer_regex": "Let's Encrypt",
  "subject_regex": "example\\.com",
  "allow_self_signed": false
}
```

Only `enabled` is required. Set `allow_self_signed` for internal endpoints with
their own CA, otherwise leave it out.

### `address_families`

Available on `http`, `http_headers`, `tcp`, `ssl_cert`, `smtp`, `imap`, `ping`
and `traceroute`.

```json
{ "address_families": ["ipv4", "ipv6"], "family_fail_severity": "degraded" }
```

Defaults to `["ipv4"]`. `family_fail_severity` applies only when both families
are checked, and says what happens when one works and the other does not.
Choosing `degraded` there is usually right, since IPv6-only failure is a real
finding but rarely a full outage.

Not every region probes IPv6. Selecting `ipv6` restricts the usable regions.
