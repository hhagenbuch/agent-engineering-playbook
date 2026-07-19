# Runbook 5 — `mcp-pact`: contract testing for MCP servers

> **Produced:** `mcp-pact`, consumer-driven contract testing for MCP servers — a versioned pact schema, a recorder, a verifier with a BREAKING/COMPAT/WARN drift taxonomy, and a CI action. **Repo:** https://github.com/hhagenbuch/mcp-pact.
> **What the review cycle caught:** silent-pass holes in the diff engine (drift that should have been flagged slipped through as a pass) and mishandled string JSON-RPC ids (the spec allows string or number ids; string ids were dropped) — both fixed (PRs #1, #4).

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `mcp-pact` — Pact-style consumer-driven contracts for MCP tool calls: record what an agent depends on, verify servers against it in CI, catch breaking changes before they silently break agents.
**Builds on:** Runbook 1. The starter's MCP client is the natural consumer for the demo but not a hard blocker — the bundled example consumer works standalone.

## The pitch (top of README)

> When an MCP server renames a tool, tightens a parameter, or subtly changes
> response shape, nothing fails at deploy time — the agents depending on it
> just quietly get worse. `mcp-pact` brings consumer-driven contract testing
> to MCP: agents record the tool interactions they rely on into a pact file;
> servers verify against those pacts in CI. Breaking changes become a red
> build, not a production mystery.

## Why this one gets stars

- The pain is brand new and universal: everyone wiring agents to MCP servers is one `tools/list` change away from silent breakage, and nobody has a testing story yet.
- The concept is instantly legible to any engineer who knows Pact — "Pact for MCP" needs no further explanation.
- It's small and sharply scoped: a schema, a recorder, a verifier, a CI action.

## Design

### The pact file (`*.mcp-pact.json`) — the real product

Versioned, documented JSON schema (`docs/SCHEMA.md` + a JSON Schema file):

```json
{
  "pactVersion": "0.1",
  "consumer": "support-agent",
  "provider": "workspace-tools-mcp",
  "expectations": [
    {
      "tool": "search_code",
      "inputSchema": { "...captured JSON Schema subset the consumer uses..." },
      "requiredCapabilities": ["tools"],
      "interactions": [
        {
          "description": "search by symbol name",
          "input": { "query": "CartService", "limit": 5 },
          "response": {
            "matchers": [
              { "path": "$.content[0].type", "equals": "text" },
              { "path": "$.content[0].text", "regex": "CartService" },
              { "path": "$.isError", "equals": false }
            ]
          }
        }
      ]
    }
  ]
}
```

Key decision (put in DESIGN.md): responses are checked with **matchers, not
literal equality** — MCP tool output is often nondeterministic, so pacts
assert shape and invariants (Pact learned this lesson a decade ago; inherit
it deliberately).

### Breaking-change rules (the verifier's brain)

Classify provider drift like semver, and print it that way:

- **BREAKING** — tool the pact uses is missing/renamed; a used input param removed or its type changed; a new **required** input param added; a matcher fails on replay; server capability withdrawn.
- **COMPAT** — new optional params, new tools, looser input schema, extra response fields.
- **WARN** — tool description changed materially (flag only — description drift changes model behavior even when schemas hold; this MCP-specific insight is worth highlighting in the README).

**No silent passes.** Every drift the diff engine can observe must resolve to
exactly one of these three verdicts; anything unclassified is a bug, not a
pass. Make the taxonomy the table-driven test suite's whole job (this was a
review finding — drift was slipping through as a pass). JSON-RPC ids may be
**string or number** per the spec; the verifier must round-trip both (another
review finding — string ids were mishandled).

### Components (one Maven multi-module repo, Java 21)

1. **`mcp-pact-core`** — pact model, JSON (de)serialization, schema-diff engine, matcher engine. Zero MCP dependency; pure, heavily unit-tested logic.
2. **`mcp-pact-recorder`** — a transparent stdio proxy: agent ↔ recorder ↔ real server. Speaks JSON-RPC 2.0 passthrough, observes `initialize`, `tools/list`, `tools/call`, accumulates expectations, writes the pact on exit. Usage: point your MCP client config at `mcp-pact record --out support-agent.mcp-pact.json -- npx some-mcp-server`. Recording real traffic instead of hand-writing contracts is what makes adoption realistic.
3. **`mcp-pact-verifier`** — CLI (shaded jar): `mcp-pact verify pact.json -- <server command>`. Launches the server over stdio (official MCP Java SDK — `io.modelcontextprotocol.sdk`; pin the current version and note MCP spec revision compatibility in the README), runs `tools/list` diff + replays interactions, prints a BREAKING/COMPAT/WARN report, exits nonzero on BREAKING. `--strict` fails on WARN too. HTTP/SSE transport is a roadmap item, not MVP.
4. **`examples/`** — a tiny in-repo MCP server (Java) + consumer pact, wired into CI as a self-test; plus a recorded pact against one well-known public MCP server (e.g. the reference filesystem server) as a realism proof.
5. **GitHub Action** (`action.yml` in-repo): servers add "verify pacts on every PR" in ~5 lines. This is the growth loop — a provider CI badge that says "consumer contracts verified".

## Build phases

### Phase 0 — Design doc (half day)

`docs/DESIGN.md`: the schema, matcher semantics, breaking-change taxonomy
(table), recorder architecture, explicit non-goals (HTTP transport, resources/
prompts contracts, pact broker — all roadmap). Commit first, alone.

### Phase 1 — core

Pact model + matcher engine + schema-diff with the full BREAKING/COMPAT/WARN
taxonomy as table-driven unit tests (this test suite *is* the spec — make it
exhaustive, ~30 cases, and include a "must not silently pass" case for every
drift shape). No I/O.

### Phase 2 — verifier

MCP Java SDK stdio client, `tools/list` diff, interaction replay, report
formatting (human + `--json`), exit codes. Cover string and numeric JSON-RPC
ids. e2e-test against the in-repo example server, including a mutated "v2" of
it that triggers each BREAKING class.

### Phase 3 — recorder

JSON-RPC passthrough proxy with observation. Trickiest bit: schema *capture*
— record the subset of the input schema the consumer actually exercised
(fields it sent), not the server's full schema; that's what "consumer-driven"
means and belongs in the README's design section. Roundtrip e2e: record
against example server → verify the recorded pact passes → mutate server →
verify fails correctly.

### Phase 4 — action + README + launch

`action.yml`, CI self-test badge, README (pitch, 60-second demo GIF of a
verify run catching a renamed tool, taxonomy table, quickstart for both
consumer and provider), `CONTRIBUTING.md`. This is the one repo worth a small
public launch (an MCP community post — "contract testing for MCP servers") —
after CI is green and the README GIF exists.

## Guardrails

No employer references anywhere, the forbidden-string grep must return 0
pre-push, HTTPS to the intended account only, design doc first, phases as
separate commits. Clean room — the design derives only from the public MCP
spec, not from any private MCP work.

## Acceptance checklist

- [ ] DESIGN.md + schema committed before code
- [ ] Taxonomy fully covered by table-driven core tests; no drift resolves to a silent pass
- [ ] String and numeric JSON-RPC ids both handled
- [ ] Verifier catches every BREAKING class against the mutated example server (e2e)
- [ ] Record → verify roundtrip green in CI; action.yml self-test badge on README
- [ ] README GIF shows a renamed tool caught in CI
- [ ] Public at github.com/hhagenbuch/mcp-pact, forbidden-string grep clean
