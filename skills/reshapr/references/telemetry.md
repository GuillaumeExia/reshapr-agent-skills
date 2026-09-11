# reShapr and OpenTelemetry

Verified 2026-09-11 against gateway `0.2.3-SNAPSHOT` (Quarkus 3.35.4, OTel Java
SDK 1.60.1) exporting straight to SigNoz Cloud, no collector in between.

## Two switches, and both are required

Telemetry and audit are separate decisions that fail in different ways:

| Switch | Where | Without it |
|---|---|---|
| `QUARKUS_OTEL_SDK_DISABLED=false` | gateway process env | nothing leaves the box at all |
| `--audit` | the **Configuration Plan** | traces and metrics flow, audit records never exist |

Shipped images ship with the SDK **off** — `QUARKUS_OTEL_SDK_DISABLED=true`, with
a comment that it avoids connection errors when no collector is running. So a
fresh gateway exports nothing until you override it. [source:
`install/docker-compose-all-in-one.yml`; the proxy chart documents the same
default]

`--audit` is set at plan creation and a plan is immutable, so decide it up front:
turning audit on later means a new plan and an exposition swap, which costs a gap
if the name must be kept.

## Enabling it

Four variables, direct to any OTLP endpoint:

```
QUARKUS_OTEL_SDK_DISABLED=false
QUARKUS_OTEL_EXPORTER_OTLP_ENDPOINT=https://<otlp-endpoint>:443
QUARKUS_OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
QUARKUS_OTEL_EXPORTER_OTLP_HEADERS=<auth-header>=<key>
QUARKUS_OTEL_RESOURCE_ATTRIBUTES=service.name=<name>,deployment.environment=<env>
```

Quarkus enables traces, metrics and logs together in the production profile;
audit rides the **logs** pipeline. A collector is only needed if you want to
route audit records separately — every audit record carries `log.type=audit`,
which is the attribute to filter on.

## What the gateway emits

### Spans

```
POST /mcp/{organizationId}/{expositionName}   kind=Server
├─ user_agent.original, client.address, expositionName, organizationId,
│  url.path, url.scheme, http.request.method, http.response.status_code
└─ ToolCallExecutor.execute                   kind=Internal   mcp.target.name
   └─ ToolCallExecutor.execute                kind=Internal   mcp.target.name   <- nested duplicate
      └─ ProxyService.doCallBackend           kind=Client     backendEndpoint
```

Also present, unrelated to MCP traffic: `reshapr.health.v1.GatewayHealthService/
AdvertHealthy` and `reshapr.exposition.v1.ExpositionDiscoveryService/*` gRPC
client spans — the gateway's own control-plane chatter. They inflate any
unfiltered span count. [verified]

The docs state the gateway continues an inbound `traceparent` and injects context
and baggage into HTTP backend calls. [doc — not verified here]

### Audit records

Scope `io.reshapr.audit`. `INFO` for success, `WARN` for failure.

| Attribute | Notes |
|---|---|
| `log.type` | always `audit` — the routing key |
| `event.action` | observed: `tools/call`, `tools/list`, `resources/list`, `prompts/list`, `server/discover`, `authentication` |
| `event.outcome` | `success` / `failure` |
| `event.reason` | on failures: `missing_bearer_token`, `malformed_token` |
| `event.duration` | gateway-side ms, **number** |
| `mcp.target.name` | tool or resource name |
| `mcp.request.id`, `mcp.session.id`, `mcp.response.size` | |
| `service.id`, `service.name`, `service.version` | the **exposed Service**, not the gateway |
| `organization.id`, `user.id`, `source.ip` | `user.id` is the raw OAuth subject; no username is emitted |
| `http.response.status_code` | on authentication failures |
| `trace.id` | correlate on this attribute — see below |

Rejected requests are audited, so authentication failures are visible without
any extra configuration — this is the useful half for security monitoring.

### Metrics

`http.server.request.duration.*`, the full `jvm.*` set, and `otel.sdk.*` exporter
self-telemetry. The presence of `otel.sdk.*` is the signal that the SDK is
running and exporting.

## Four things to know when querying

1. **Correlate audit records on the `trace.id` attribute.** The trace id travels
   as an attribute rather than in the intrinsic `trace_id`/`span_id` log columns,
   so a backend's "logs for this trace" view will not pick these records up on
   its own. Filter on the attribute and the link is there. [verified]

2. **`service.name` appears in two contexts.** As a *resource* attribute it names
   the gateway process; as a log or span *attribute* it names the exposed
   Service. Both are useful; qualify which one a filter means
   (`resource.service.name` vs `attribute.service.name`), since a backend will
   otherwise pick a default for you. [verified]

3. **The user agent and the tool name sit on different spans.**
   `user_agent.original` is on the HTTP server span, `mcp.target.name` on its
   child. "Which client called which tool" is therefore a parent/child join
   rather than a single query, and it is a traces question — audit records do
   not carry a user agent. [verified]

4. **Join those spans parent-to-child, not on trace id.** Each call produces a
   nested pair of `ToolCallExecutor.execute` spans, and one trace can carry
   several HTTP spans, so a trace-id join multiplies rows. Matching
   `child.parent_span_id = server.span_id` lines up exactly with the audit-log
   counts, which is the reconciliation worth keeping. [verified]

## Verifying the pipeline

- **Check `otel.sdk.*` metrics in the backend, not `/q/metrics`.** Once the SDK
  is on, `/q/metrics` returns `200` with an empty body, so it no longer signals
  whether export is working. [verified — the empty response is observed; the
  reason for it is inference, not established]
- Generate **both** outcomes: an unauthenticated call (expect `401` and one
  `event.outcome=failure` record) and an authenticated `tools/call` (expect
  `success`, with `mcp.target.name` and `event.duration`).
- Assert on record content, not on "logs arrived". Control-plane chatter and
  health spans arrive whether or not MCP traffic works.
