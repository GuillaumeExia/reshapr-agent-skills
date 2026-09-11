---
name: reshapr
description: Use when working with reShapr — creating or changing MCP expositions, configuration plans, Services, CustomTools artifacts, cross-service scripted tools, backend secrets, gateway telemetry and audit, or the `reshapr` CLI. Field notes from running reShapr 0.2.3 — the conventions, ordering and verification steps that get an exposition working first time. Triggers on "reShapr", "exposition", "CustomTools", "config plan", "rs.callTool", "mcp/reshapr", "--audit", reShapr audit records or gateway OpenTelemetry, or any reshapr-cli command.
---

# reShapr

Working notes from building and operating MCP expositions on reShapr
`0.2.3-SNAPSHOT` with CLI `0.2.1`. Claims carry their provenance:
**[verified]** (observed by running it), **[source]** (read in the 0.2.3 tree at
`github.com/reshaprio/reshapr`), **[doc]** (from the official documentation) or
**[unverified]**.

Behaviour moves between releases, so re-test after an upgrade rather than
assuming these notes still hold.

- `references/cli.md` — commands, flags, session handling
- `references/telemetry.md` — OpenTelemetry: enabling it, what the signals
  contain, and how to query them

## Object model and its constraints

```
Service ──< Artifact (OPEN_API_SPEC, RESHAPR_CUSTOM_TOOLS, …)
   └──< ConfigurationPlan ──< Exposition ──> GatewayGroup
```

- A Service is one OpenAPI document. Its identity is `info.title` +
  `info.version`, **fixed at import** — changing either creates a *second*
  Service rather than updating one, so settle the title and version before the
  first import. [verified]
- A plan references exactly one Service; an exposition references exactly one
  plan; **a plan carries one exposition**. For a second endpoint, create a
  second plan. [verified]
- A Service may hold *several* plans. That is the lever for gapless renames.
- Exposition names are organization-unique, and an exposition is created rather
  than updated.
- A plan is likewise immutable in CLI 0.2.1: changing `--bt`, the backend or the
  filters means creating a new plan and moving the exposition to it. Plan for
  that rather than around it.

## Before you build

1. **Check the backend accepts what the gateway will send.** Array *query*
   parameters arrive comma-joined, so the backend must accept `?p=A,B` as well
   as `?p=A&p=B`. A backend that reads `A,B` as a single value returns nothing,
   so confirm this with two values early. Request bodies are serialized as JSON
   and arrays, including arrays of objects, survive intact. [source + verified]
2. **Derive the generated tool names** and check for collisions across every
   Service you will expose together:
   `name.replace(" ","_").replace("/","_").replace("{","").replace("}","").replace("__","_").toLowerCase()`
   applied to `"GET /articles"`. Names come from method and path. [source]
3. **Measure latency, including a cold start**, and set `--bt` accordingly. The
   default is 3000 ms and it is fixed at create.
4. **Decide the auth shape.** `ProxyService` offers two: no `tokenHeader` →
   `Authorization: Bearer <token>`; `tokenHeader` set → `<header>: <token>` sent
   raw, with no scheme prefix. A non-Bearer scheme therefore goes inside the
   token value: `secret create x -B -t 'ApiKey <token>' -h Authorization`.
   [source + verified]

## CustomTools artifacts

Plan to override **every** tool. Generated names are machine-derived, and a
path-derived name carries path content into the tool name — worth checking when
a path contains something like a webhook UUID. Generated tools carry the
operation's `description`, falling back to `summary`, and no `outputSchema`.

Two conventions to get right:

- **`tool:` takes the OPERATION name** (`GET /articles`), not the generated tool
  name (`get_articles`). Copy it verbatim from `service get` and compare by
  string equality rather than by eye.
- **`arguments:` on every declarative tool** — `{}` when the tool takes no
  parameters.

Scripted tools need neither, because `getCallResponse` returns via
`executeScript` before the arguments template is read.

Leave `--ia` off: scoping a plan to the CustomTools artifact alone excludes the
OpenAPI artifact the overrides point at. Hide the generated duplicates with the
plan's `--eo`, listing operation names.

The description is the only channel for semantics. Write the caveats a caller
needs in order to read a result correctly, not a restatement of the name.

## Cross-service scripted tools

A scripted tool may call another Service in the same organization:

- The far Service's backend endpoint and secret are applied **automatically** —
  nothing to configure on the calling side. [verified]
- **The far Service needs an exposition**, since resolution goes through its
  elected exposition. Create one even if nobody will connect to it directly.
  [verified]
- **The far plan's `--eo` filter applies to the script's call too**, so target
  the far service's custom tool name rather than an operation that plan
  excludes. [source + verified]
- `rs.callTool` returns `{ok, content, error}`. A far backend 4xx gives
  `ok:false` with the parsed error body in `content`; gateway-level outcomes
  (allow-list, unknown service, elicitation) populate `error`. Nothing throws,
  so branch on `ok` explicitly. [verified]

**Reporting failure from a script.** `executeScript` reports success unless the
script itself fails, so a wrapper that returns `r.content` unconditionally will
present a far 4xx as a good result. Use **`rs.fail(message, data)`**: it yields
`{"message": "...", "data": {...}}`, giving a human message plus the far
service's machine-readable code. It is the clean path and worth preferring over
a raw `throw`. [verified]

Guard-rails: script timeout **10 s**, max **10** tool calls per script, max
nesting depth **5**. `--bt` does not extend the script timeout, so keep slower
backends declarative and build composites over fast services.

A wrapper does not inherit the far tool's `description` or `input`, so generate
both from the source artifact rather than transcribing them.

## Operating expositions

Confirm registration with an uncredentialed request — a registered exposition
answers `401`, an unknown one `404`:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST <endpoint> \
  -H 'content-type: application/json' -H 'Mcp-Method: tools/list' -d '{}'
```

**Renaming to a free name — no gap needed.** A plan carries one exposition but a
Service holds many plans, so stand the new one up beside the old: create a
second plan with identical settings, create the new exposition on it, confirm
`401`, then remove the old exposition and its plan. Both serve at once.

**Keeping the name while changing settings** means a brief gap, since names are
unique. Create the new plan first, run the delete and create back-to-back,
verify immediately, and keep the old plan until the replacement is confirmed.

**If a new exposition stays 404.** Registration normally completes in under ten
seconds. The gateway learns about expositions at startup and then follows a
changes stream, so a 404 persisting past a minute is a registration that has not
arrived rather than slowness — restart the gateway and it picks everything up
immediately. Pre-existing expositions keep serving throughout. [verified]

## Telemetry and audit

Two independent switches, and both are needed. The gateway ships with
`QUARKUS_OTEL_SDK_DISABLED=true`, so enable the SDK for anything to be exported;
`--audit` on the Configuration Plan decides whether audit records are produced
at all. Audit flows through the OpenTelemetry **logs** pipeline — scope
`io.reshapr.audit`, every record tagged `log.type=audit`.

Since a plan is immutable, `--audit` is a create-time decision: adding it later
means a new plan and an exposition swap.

Enabling takes four env vars on the gateway and no collector — it exports OTLP
directly. Note that `/q/metrics` returns an empty `200` once the SDK is on, so
check for `otel.sdk.*` metrics in your backend instead. The audit attribute set,
the span tree and the querying notes are in `references/telemetry.md`.

## Verification discipline

Every check should prove its own claim.

- **A 401 alone establishes little.** It is returned identically for a missing
  credential, a wrong one, and an unrecognised header name.
- **Assert on payload content, never byte counts.** An empty `{}` has a length.
- **Enforce input limits in the backend.** A `pattern` or `maxItems` in a
  CustomTools `input` guides the model; the backend is what makes it binding.
  [verified]
- **A tool being listed says nothing about calling it.** Call it.
- After re-attaching an artifact the gateway serves the new content immediately,
  but **MCP clients cache tool schemas across a reconnect**. Verify from a
  client that has never fetched that server, or restart the client. [verified]
- Design demos so unsafe cases are impossible rather than merely invalid. A
  capable model reads a documented limit and routes around it.

## After an upgrade

Which custom tools are exposed, and how that interacts with a plan's `--eo`
filter, is an area under active development upstream. Re-run `tools/list` and
call one tool after any gateway upgrade rather than assuming the previous
arrangement carried over. [source]
