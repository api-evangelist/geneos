---
name: geneos-run-gateway-command
description: >-
  Run a Geneos command (snooze, unsnooze, reload setup, user-defined commands)
  against a Gateway over its REST command service, safely — rehearse the target
  first, then act.
api: geneos:gateway-rest
generated: '2026-09-12'
method: generated
source: >-
  https://docs.itrsgroup.com/docs/geneos/current/processing/monitoring-and-alerts/geneos_commands_tr/index.html#rest-service
operations:
  - 'POST /rest/runCommand'
  - 'POST /rest/runCommandOnMultipleTargets'
  - 'GET /rest/commands/all'
  - 'POST /rest/commands/available'
  - 'POST /rest/xpaths/match'
  - 'POST /rest/xpaths/commandTargets'
  - 'POST /rest/snapshot/dataview'
  - 'GET /rest/setup/validate'
  - 'GET /rest/gatewayinfo/timezone'
  - 'GET /rest/authorize'
---

# Run a command on a Geneos Gateway

**There is no OpenAPI for this surface.** Every endpoint, body field and status code
below is quoted from ITRS Group's Gateway Commands technical reference. Nothing has
been generated into a contract that ITRS does not publish.

## This is the highest-consequence surface in Geneos

`/rest/runCommand` executes arbitrary named Geneos commands. The documented set
includes `/GATEWAY:reloadSetup` and `/RMS:rollback` (roll a Netprobe back to an
older version). It is **not idempotent** — replaying a request repeats the side
effect — and there is **no generic undo**. Some commands have a specific inverse
(`/SNOOZE:manual` ↔ `/SNOOZE:unsnooze`); most do not.

Rehearse before you act.

## 0. Check the service is even on

The REST service is **disabled by default** (`commands > restService > enabled`
defaults to false). A `404` on any of these endpoints means one of two things and
the response does not tell you which:

- the Gateway is not running a REST service at all, or
- the REST service is secure and you called the insecure port.

## 1. Authenticate

Two options, both documented:

```
# HTTP Basic, for users with passwords in the Gateway setup
curl -u rest_user:PASSWORD ...

# SSO bearer token
SSO_TOKEN=$(curl -L --location-trusted --negotiate --user : -X GET -s -N \
  http://{gatewayHost}:{restPort}/rest/authorize | jq -r '.access_token')
curl -H "Authorization: Bearer $SSO_TOKEN" ...
```

System logins are **not supported** for REST. ITRS notes the Basic header is
base64-encoded but not encrypted, and recommends restricting the REST service to
secure connections whenever authentication is on.

## 2. Rehearse — the closest thing Geneos has to a dry run

There is no `dryRun` flag. There are three read-only endpoints, and using them is
the documented way to avoid a failed or mis-targeted command:

```
# Which commands exist at all? (unfiltered — ignores permissions)
GET /rest/commands/all

# Which commands can I run on this target? (filtered by my permissions)
POST /rest/commands/available
{ "target": "/geneos/gateway[(@name=\"GW\")]/directory/probe[(@name=\"P\")]",
  "namePattern": "^/SNOOZE:[mi]" }

# Which items would this command hit? (filtered by my permissions)
POST /rest/xpaths/commandTargets
{ "target": "//managedEntity", "command": "/SNOOZE:unsnooze" }
```

Use `/rest/commands/available` and `/rest/xpaths/commandTargets`, not
`/rest/commands/all` and `/rest/xpaths/match` — the first pair respects the calling
user's permissions, the second pair deliberately does not, so the second pair will
show you things you cannot actually do.

## 3. Act

```
POST /rest/runCommand
{
  "command": "/SNOOZE:manual",
  "target": "/geneos/gateway[(@name=\"GW\")]/directory/probe[(@name=\"P\")]/managedEntity[(@name=\"ME\")]",
  "args": { "1": "Remedial work in progress", "2": 3, "4": 1 }
}
```

- `command` — required. The command name.
- `target` — required. A Geneos XPath. It need not be fully qualified, but it must
  resolve to **exactly one** item or you get
  `{"error": "Command target matches more than one item"}`.
- `args` — a numbered map, present only for commands that take arguments. A command
  invoked through REST with the wrong number of arguments fails; it cannot prompt
  the way Active Console can.

Success is `{"status": "finished"}`. For multiple targets use
`/rest/runCommandOnMultipleTargets`, which returns a per-target array plus an
overall status.

## 4. Streaming output

Send `Accept: text/event-stream` to receive command output as Server-Sent Events:
a `mimetype` message, then `stdout` / `stderr` / `execLog` events, then
`{"status":"finished"}` or `{"status":"error"}`.

Two things will catch you out:

- **The Gateway does not close the connection.** Your client must close it after
  seeing the status message.
- **Do not auto-reconnect.** ITRS states that the Gateway does not support
  reconnecting to a command output stream: a dropped connection cancels the command
  if it has not finished, and a client that reconnects **reruns the command**. A
  naive SSE retry loop will execute your command twice.

## Read-only things worth doing instead

```
POST /rest/snapshot/dataview     # values, severity and snooze status of one dataview
GET  /rest/setup/validate        # validation report for the running setup
GET  /rest/gatewayinfo/timezone  # timezone and UTC offset, so you can build correct
                                 # date/time arguments without guessing the difference
```

`/rest/setup/validate` requires setup view (or apply) permission when
authentication is enabled.

## Errors

| Status | Meaning |
|---|---|
| `200` | Command ran successfully. |
| `400` | Body is not well-formed JSON, or a mandatory item is missing. |
| `403` | Invalid credentials **or** valid credentials without permission for that command on that target — the response does not distinguish them. |
| `404` | REST service not running, or secure service called on the insecure port. |
| `500` | The command itself returned an error. |

Failures return `{"error": "<message>"}`. Full catalogue in
`errors/geneos-problem-types.yml`.
