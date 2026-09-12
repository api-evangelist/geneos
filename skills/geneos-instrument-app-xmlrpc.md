---
name: geneos-instrument-app-xmlrpc
description: >-
  Instrument an in-house application so it publishes its own custom dataviews and
  log streams into Geneos over the XML-RPC Instrumentation API, with heartbeat
  monitoring so a dead publisher raises an alert.
api: geneos:xml-rpc
generated: '2026-09-12'
method: generated
source: https://docs.itrsgroup.com/docs/geneos/current/collection/xml-rpc-api/index.html
operations:
  - 'entity.sampler.createView(string viewName, string groupHeading)'
  - 'entity.sampler.viewExists(string groupHeading-viewName)'
  - 'entity.sampler.removeView(string viewName, string groupHeading)'
  - 'entity.sampler.getParameter(string parameterName)'
  - 'entity.sampler.view.addTableColumn(string columnName)'
  - 'entity.sampler.view.addTableRow(string rowName)'
  - 'entity.sampler.view.updateTableCell(string cellName, string newValue)'
  - 'entity.sampler.view.updateTableRow(string rowName, array newValue)'
  - 'entity.sampler.view.updateEntireTable(array newTable)'
  - 'entity.sampler.view.removeTableRow(string rowName)'
  - 'entity.sampler.view.addHeadline(string headlineName)'
  - 'entity.sampler.view.updateHeadline(string headlineName, string newValue)'
  - 'entity.sampler.view.removeHeadline(string headlineName)'
  - 'entity.sampler.signOn(int seconds)'
  - 'entity.sampler.heartbeat()'
  - 'entity.sampler.signOff()'
  - 'entity.sampler.stream.addMessage(string message)'
  - '_netprobe.managedEntityExists(string managedEntity)'
  - '_netprobe.samplerExists(string sampler)'
  - '_netprobe.gatewayConnected()'
  - '_gateway.addManagedEntity(string managedEntity, string dataSection)'
---

# Instrument an application with the Geneos XML-RPC API

Every method name below is quoted from ITRS Group's published API Technical
Reference. The transport is plain XML-RPC over HTTP (or HTTPS when the Netprobe
runs in secure mode), on the same port the Gateway connects to. Any XML-RPC client
in any language works; ITRS ships first-party Go bindings in
[`cordial`](https://github.com/ITRS-Group/cordial) under `pkg/geneos/xmlrpc`,
`pkg/geneos/samplers` and `pkg/geneos/streams`.

## Two things to settle before writing code

**Licensing.** "The API functionality is separately licensable. Please contact ITRS
Sales for details." This is not enabled by default. Confirm it before building
against it.

**Security.** "XML-RPC traffic is neither encrypted or authenticated." There is no
credential to present. Access control is a host allow-list: `TRUSTED_API_HOSTS`,
set as an environment variable or in the managed entity descriptor, listing trusted
hosts and IPs. Calls from anywhere else return `HOST_NOT_TRUSTED` immediately and
the first such call per host is logged. ITRS suggests IPSec where confidentiality
matters. Design accordingly — do not put secrets in dataview cells.

## Method naming

The method name *is* the address:

```
<ManagedEntity>.<Sampler>.<function>              # sampler-scoped
<ManagedEntity>.<Sampler>.<group><view>.<function> # view-scoped
<ManagedEntity>.<Sampler>(<Type>).<function>       # when the sampler name is ambiguous
```

`_netprobe` and `_gateway` are reserved pseudo-entities. Never name a managed entity
either of those.

## Steps

### 1. Verify the target exists before publishing

```
_netprobe.gatewayConnected()                 -> boolean
_netprobe.managedEntityExists("myManEnt")    -> boolean
_netprobe.samplerExists("myManEnt.mySampler")-> boolean
```

These return booleans rather than the usual `"OK"` string, and they have no side
effects. Call them on startup and after any reconnect — a Netprobe restart leaves
the sampler in place but destroys the views, which is exactly what
`entity.sampler.viewExists` is documented to detect.

### 2. Create the view

```
myManEnt.mySampler.createView("myView", "myGroup")
```

The group heading **must differ from the view name** (`VIEW_AND_GROUP_EQUAL`
otherwise — ITRS calls the constraint historical). View names must be unique within
and across groups; duplicates make rules misbehave.

The view appears in Active Console as soon as it is created, even while empty.

### 3. Build the table

Columns first, then rows, then cells:

```
myManEnt.mySampler.myGroupmyView.addTableColumn("latency_ms")
myManEnt.mySampler.myGroupmyView.addTableRow("order-gateway-1")
myManEnt.mySampler.myGroupmyView.updateTableCell("order-gateway-1.latency_ms", "12")
```

For bulk updates prefer `updateTableRow(rowName, array)` or
`updateEntireTable(array)` over cell-at-a-time calls.

Headlines are scalars above the table:

```
myManEnt.mySampler.myGroupmyView.addHeadline("lastRunId")
myManEnt.mySampler.myGroupmyView.updateHeadline("lastRunId", "4417")
```

### 4. Commit to a heartbeat so silence raises an alert

This is the part most integrations skip and then regret.

```
myManEnt.mySampler.signOn(120)   # "I will update at least every 120 seconds"
myManEnt.mySampler.heartbeat()   # call when you have no data but are alive
myManEnt.mySampler.signOff()     # when you deliberately stop
```

`signOn` accepts 1 to 86400 seconds (`NUMBER_OUT_OF_RANGE` outside that). Miss the
window and every view belonging to the sampler flips its `samplingStatus` headline
to `FAILED: No heartbeat from the client in the last xx seconds`, which is what you
set a rule on.

`signOff` and `heartbeat` when not signed on return `"OK"` and do nothing — safe to
call blind. `signOn` again to change the interval; no need to sign off first.

Pair this with **Expect Views** in the plug-in setup: heartbeats catch a publisher
that *stopped*, Expect Views catch a publisher that *never started*.

### 5. Write log lines to a stream

```
myManEnt.myStreamsSampler.a.addMessage("This is a line to be sent through a stream to FKM")
```

A consuming sampler (FKM, Trapmon, TIB-RV Stream, Windows Event Queue) reads the
stream; once read, lines are gone. The buffer defaults to 1000 messages and drops
the earliest on overflow, counting them in `totalMessagesLost`. If no consumer is
attached, "the stream registry purges the messages immediately" — a stream with no
reader is a black hole, not a queue.

FKM references a stream as `<ManagedEntity>.<Sampler>.<Stream>`, with the managed
entity part omittable when FKM and the stream sampler share one.

## Idempotency and reversal

The `update*` methods are last-write-wins upserts and are safe to replay. The
`add*`/`create*` methods are **not** — replaying one returns `VIEW_EXISTS`,
`ROW_EXISTS`, `COLUMN_EXISTS` or `HEADLINE_EXISTS`. `addMessage` appends every time.

Reversals exist (`removeView`, `removeTableRow`, `removeHeadline`, `signOff`) and
**no time window is documented for any of them**. `removeView` additionally fails
with `GATEWAY_NOT_SUPPORTED` while a Gateway 1 is connected. Stream messages cannot
be recalled.

## Errors

All functions return a value — `"OK"` on success where there is no specific return.
On failure you get a named code and description. The full catalogue is in
`errors/geneos-problem-types.yml`; the ones you will hit in normal operation are
`GATEWAY_NOT_CONNECTED`, `NO_SUCH_SAMPLER`, `NO_SUCH_VIEW`, `LIMIT_REACHED`
(200-row default ceiling, `MAX_GM_ROWS`), `STREAM_BUFFER_FULL` and
`HOST_NOT_TRUSTED`.

Treat `GATEWAY_NOT_CONNECTED` as retryable and `HOST_NOT_TRUSTED` as a
configuration failure that will never resolve on its own.
