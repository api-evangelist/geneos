---
name: geneos-publish-dataview
description: >-
  Publish a table of application data into ITRS Geneos monitoring as a custom
  dataview, using the Netprobe REST API plug-in, and clean it up afterwards.
api: geneos:netprobe-rest-api
generated: '2026-09-12'
method: generated
source: openapi/geneos-netprobe-rest-api-openapi.yml
operations:
  - 'PUT /v1/managedEntity/{me}/sampler/{sampler}(type)/dataview/{dataview}'
  - 'PUT /v1/managedEntity/{me}/sampler/{sampler}(type)/dataview/{dataview}/row/{row}'
  - 'DELETE /v1/managedEntity/{me}/sampler/{sampler}(type)/dataview/{dataview}/row/{row}'
  - 'DELETE /v1/managedEntity/{me}/sampler/{sampler}(type)/dataview/{dataview}'
  - 'GET /v1/healthcheck'
---

# Publish a dataview to Geneos

ITRS Group's published OpenAPI declares no `operationId` on any operation, so every
step below is addressed by method and path exactly as they appear in
`openapi/geneos-netprobe-rest-api-openapi.yml`. Nothing here is invented.

## Before you start

Geneos is not a hosted service. The base URL is a Netprobe **inside the estate you
are working in**: `http://{netprobeHost}:7136/v1`, or `https://{netprobeHost}:7137/v1`
when the Netprobe was started in secure mode with `-secure`, `-ssl-certificate` and
`-ssl-certificate-key`.

Three preconditions, all of which fail loudly if unmet:

1. A sampler using the `rest-api` plug-in exists on the target Netprobe.
2. The **managed entity** and **sampler** already exist in the Gateway setup. Neither
   API creates them. A `404` on a write usually means one of these is missing, not
   that the dataview is missing.
3. The REST API plug-in is licensed — it requires a GA4.12.x Netprobe and a GA4.12.x
   Licence Daemon.

Confirm reachability first:

```
GET /v1/healthcheck
```

`200` means the plug-in is listening. `500` means it is not healthy — stop and
escalate; do not start writing.

## Naming rules that will bite you

- Dataview names must be unique **within and across groups** on a sampler. The spec
  says so in its own description: "resource names must be unique. Avoid giving your
  samplers and dataviews the same name."
- If two samplers share a name under different Types, put the type in parentheses:
  `/sampler/mySampler(myType)/`. Omit it when the name is unambiguous.

## Steps

### 1. Create or replace the dataview

```
PUT /v1/managedEntity/{me}/sampler/{sampler}/dataview/{dataview}
Content-Type: application/json

[ { "column": "value", ... }, { "column": "value", ... } ]
```

The body is the `GeneosDataview` schema: a JSON array of objects, one per row. The
first object's keys become the columns.

This is an **upsert**. Sending it twice with the same body leaves the same state —
that is the only replay protection Geneos offers, and it comes from the HTTP verb,
not from any Idempotency-Key mechanism (there is none anywhere in Geneos).

Declared responses: `200`, `400`, `404`, `500`.

### 2. Update individual rows as data changes

```
PUT /v1/managedEntity/{me}/sampler/{sampler}/dataview/{dataview}/row/{row}
```

Also an upsert. Prefer this over re-PUTting the whole dataview for incremental
updates — a whole-dataview PUT replaces the table.

### 3. Watch the row ceiling

Every custom dataview carries a `totalRows` headline and a `samplingStatus`
headline. There is a **200-row default limit** (sampler `Row limit`, or the
Gateway-wide `Operating environment > Custom dataview max rows`). Exceed it and
Geneos silently truncates the table and writes a warning into `samplingStatus` — it
does not fail your request. If your data can exceed 200 rows, either raise the limit
deliberately or aggregate before publishing. This limit has been on by default since
Gateway/Netprobe 7.7.0.

### 4. Clean up

```
DELETE /v1/managedEntity/{me}/sampler/{sampler}/dataview/{dataview}/row/{row}
DELETE /v1/managedEntity/{me}/sampler/{sampler}/dataview/{dataview}
```

Both are declared with a `200` only — the contract does not say what a delete
against something that does not exist returns.

## Reversibility

A dataview delete reverses a dataview create, and a row delete reverses a row
create. **No time window is documented for either**, and nothing is being asserted
here about one.

What is *not* reversible is the monitoring consequence. A Geneos dataview is live
state that rules run against. If you publish a row that trips a critical rule, the
alert has already fired by the time you delete the row — deleting it does not
un-alert anyone. Treat publishing into a production Gateway as an action with
downstream effects on people, not just on a table.

## Errors

| Status | What it actually means |
|---|---|
| `400` | Body is not well-formed, or a mandatory field is missing. No schema is attached to this response in the contract. |
| `404` | Declared on the dataview and row PUTs. In practice: the managed entity or sampler does not exist in the Gateway setup. |
| `500` | Declared on the PUTs and on `/healthcheck`. |

The error responses carry no schema, no description and no example, so the status
code is the whole machine-readable signal. See
`errors/geneos-problem-types.yml`.

## Security

The published contract declares **no securitySchemes and no security requirement**.
A client generated from it sends unauthenticated requests. If the sampler has
`Verify client certificate` enabled with a `Client CA certificate` path, present a
client certificate (mutual TLS). If it does not, anything that can reach port 7136
can write to and delete these dataviews — which is worth raising with whoever owns
the Netprobe.
