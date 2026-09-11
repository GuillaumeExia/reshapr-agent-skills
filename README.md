# reShapr Agent Skills

An [Agent Skill](https://code.claude.com/docs/en/skills) for working with
[reShapr](https://reshapr.io) — the MCP gateway that exposes OpenAPI services as
MCP tools.

These are field notes from building and running three expositions on 0.2.3:
the conventions, the ordering that matters, and the checks worth keeping. The
aim is that an agent picks up the working patterns straight away instead of
rediscovering them.

## What it covers

- **The object model and its constraints** — how Services, plans and expositions
  relate, which properties are fixed at create time, and how to plan around that.
- **CustomTools artifacts** — which name belongs in `tool:`, the fields every
  declarative tool needs, and how to scope a plan so overrides and generated
  tools coexist cleanly.
- **Cross-service scripted tools** — what the gateway wires up for you, the
  guard-rails to design within, and how to report failure from a script so a
  caller can act on it.
- **Operating expositions** — verifying registration, and renaming with or
  without a gap.
- **Telemetry and audit** — enabling OpenTelemetry, what each signal contains,
  and how to query the audit stream.

## Install

Copy the skill into your skills directory:

```bash
# user-level, available in every project
git clone https://github.com/GuillaumeExia/reshapr-agent-skills
cp -r reshapr-agent-skills/skills/reshapr ~/.claude/skills/

# or project-level
cp -r reshapr-agent-skills/skills/reshapr /path/to/project/.claude/skills/
```

It loads on mention of reShapr, expositions, configuration plans, CustomTools,
`rs.callTool`, `--audit`, or any `reshapr` CLI command.

## How claims are marked

Every non-obvious claim carries its provenance, so a reader can re-test rather
than take it on trust:

| Tag | Means |
|---|---|
| `[verified]` | observed by running it against a live deployment |
| `[source]` | read in the 0.2.3 tree at `github.com/reshaprio/reshapr` |
| `[doc]` | stated by the official documentation, not independently checked |
| `[unverified]` | believed, but deliberately not tested |

Written against server `0.2.3-SNAPSHOT` and CLI `0.2.1`. reShapr is moving
quickly, so treat these as notes for that version and re-test after an upgrade.

## Contributing

Corrections welcome, particularly ones that retire a note because a newer release
made it unnecessary — those pull requests are the best kind, and saying which
version changed things helps everyone else. New claims should arrive with the
evidence tag that fits, and `[verified]` should mean you ran it.

## Licence

MIT. See `LICENSE`.
