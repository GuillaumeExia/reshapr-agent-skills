# reshapr CLI 0.2.1 — commands and flags

```bash
CLI="npx -y @reshapr/reshapr-cli@0.2.1"
```

## Session

The session token **expires frequently**. Commands report it distinctly:

```
⚠️  Your token has expired. Please login again using the `reshapr login` command.
```

That is not a permissions problem — re-run `reshapr login`. `login` is
interactive, so in a non-TTY context ask the operator to run it.

Distinguish it from a genuine failure, which looks like
`❌ Failed to create exposition: Internal Server Error`.

## Read-only orientation

```bash
$CLI info                       # user, org, control plane, server version
$CLI service list
$CLI service get <id>           # operation names — the values `tool:` must contain
$CLI artifact list -s <serviceId>
$CLI artifact get <id>
$CLI config list
$CLI config get <id>
$CLI expo list                  # add -o json for names and endpoints
$CLI expo get <id>
$CLI quotas                     # exposition.count, gateway.count, gateway-group.count
$CLI gateway-group list
```

There is **no** `gateway` command and **no** refresh/reload for an exposition.

## Secrets

```bash
$CLI secret create <name> -B -t '<token>'                     # Authorization: Bearer <token>
$CLI secret create <name> -B -t 'ApiKey <token>' -h Authorization   # Authorization: ApiKey <token>
$CLI secret create <name> -B -u <user> -p <pass>              # Authorization: Basic …
$CLI secret list
```

Flags: `-A` artifact secret, `-B` backend secret, `-t/--token`,
`-h/--tokenHeader` (any header name; the value is sent **raw**, no scheme
prefix), `-u/--username`, `-p/--password`, `-c/--certificate`.

Reading a secret back may be blocked by tooling policy; do not design a workflow
that depends on inspecting one. If the scheme is wrong the only symptom is a
bare 401 from the backend, indistinguishable from several other faults — so the
first successful end-to-end call is what proves a secret.

## Build order

```bash
$CLI secret create <name> -B -t '<token>'
$CLI import -f <spec>.yaml -o json          # creates the Service; identity fixed here
$CLI service get <serviceId>                # copy operation names VERBATIM
$CLI attach -f <custom-tools>.yaml -o json  # Service resolved from the file, see below
$CLI config create-oauth "<name>" \
  -s <serviceId> \
  --be <backend base url> --bs <secretId> --bt <measured ms> \
  --eo '["GET /path", ...]' \
  --oas '["https://auth.example/realms/<realm>"]' \
  --oju https://auth.example/realms/<realm>/protocol/openid-connect/certs \
  --osc '["mcp:tools"]' \
  --audit -o json
$CLI expo create -c <configId> -g 1 -n <name> -o json
```

`config create-oauth` flags: `-s/--serviceId`, `-d/--description`,
`--be/--backendEndpoint`, `--bs/--backendSecret`, `--bt/--backendTimeout`,
`--io/--includedOperations`, `--eo/--excludedOperations`,
`--ia/--includedArtifacts`, `--oas`, `--oju`, `--osc`, `--audit`.

- `--bt` applies **only at create**. Default is 3000 ms.
- Omit `--ia` — scoping to the CustomTools artifact alone breaks every call.
- `--eo` takes operation names (`GET /articles`), not generated tool names.

## import

Re-importing a spec whose `info.title` + `info.version` already match a Service
updates that Service in place — same id, same `createdOn`, same operation list —
but **replaces its main artifact with a new id**. The old artifact is gone, not
superseded, and nothing in the control plane announces the swap. [verified]

Harmless for plans and expositions, which reference the Service, and for
CustomTools artifacts, which keep their own ids. It is only harmless for
declarative `tool:` targets **while the operation names are unchanged** — rename
a path and every override silently points at an operation that no longer exists,
which lists fine and 500s on call. Compare the operation list the import echoes
against the `tool:` values, then call a tool rather than listing one.

Nothing re-imports on its own, so the stored spec drifts behind the source
repository with no signal and can be several commits stale. Diff before
importing:

```bash
$CLI artifact get <id> -o json     # .content is the stored document
```

## attach

`attach` has **no `--serviceId`**. It resolves the target from the `service:
{name, version}` block inside the artifact, and both values must match the
Service exactly. A file without that block fails with a bare
`Attach failed: Internal Server Error`.

Re-attaching a file with the same name **updates the artifact in place, keeping
its id** — unlike `import`, which replaces the main artifact — and the change
does reach the gateway. Client-side tool-schema caches are a separate problem —
see SKILL.md.

## Deletes

```bash
$CLI expo delete <id>          # no prompt
$CLI config delete <id> -f     # -f required in a non-TTY shell
$CLI artifact delete <id>      # WARNS it "cleans up referencing configuration plans"
```

`artifact delete` cascades: its help notes that it cleans up referencing
configuration plans, so it can remove more than the artifact. Re-attaching a file
with the same name updates the artifact in place, which is usually what you want.
[unverified — the cascade is described in the CLI help, deliberately not tested]

## Local development

`$CLI run`, `$CLI status`, `$CLI stop` drive a local Docker Compose stack.
`switch-org <target>` changes organization. `api-token` manages API tokens.

## Gateway configuration worth knowing

From `proxy/src/main/resources/application.properties` at 0.2.3:

| Property | Default |
|---|---|
| `reshapr.gateway.backend.http.default-timeout` | 3000 ms |
| `reshapr.gateway.scripting.timeout` | 10000 ms |
| `reshapr.gateway.scripting.max-tool-calls` | 10 |
| `reshapr.gateway.scripting.max-depth` | 5 |
| `reshapr.gateway.mcp.cache.size` | 1000 (Caffeine, no TTL) |
| `quarkus.http.limits.max-body-size` | 10M |

Audit (`--audit` on the plan) is emitted to **OpenTelemetry**, not stdout:
`Audit logger initialized with OpenTelemetry`. The OTLP exporter endpoint is
commented out in the shipped configuration and the SDK itself ships disabled
(`QUARKUS_OTEL_SDK_DISABLED=true`), so `--audit` alone produces nothing
observable. See `telemetry.md`.
