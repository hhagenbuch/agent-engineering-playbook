# Runbook 6 — `agent-blackbox`: a flight recorder for agents

> **Produced:** `agent-blackbox`, a flight recorder for agent sessions — record every session to a durable trace, replay it deterministically, diff trajectories across versions, and export any trace into a regression eval case. **Repo:** https://github.com/hhagenbuch/agent-blackbox.
> **What the review cycle caught:** replay wasn't actually wired to the agent (it played a scripted path rather than driving the real loop) and its safety default was inverted — it executed real side-effects unless a tool was explicitly stubbed. Both were rewritten: replay now runs the real agent loop, and the safe default is no side-effects unless explicitly allowed (PR #4).

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `agent-blackbox` — record every agent session to a durable trace, deterministically replay it without touching the real model or real side-effects, diff trajectories across versions, and export any trace into an eval golden case with one command.
**Builds on:** the agent runtime (instrumented target) and the eval harness (export destination). Both are shipped; this closes the loop between them.

## The pitch (top of README)

> When an agent misbehaves in production, you get a support ticket and a shrug.
> There's no stack trace for "it chose the wrong tool and then lied about it."
> `agent-blackbox` is the flight recorder: every session is captured as an
> append-only trace — messages, tool calls, model/provenance, timings — that you
> can **replay deterministically** (no API key, no side-effects), **diff**
> against another run, and **export as a regression eval** so the same failure
> can never ship again.

The one-liner that sells it: **every production incident becomes a permanent
eval case with one command.** Debugging and testing stop being separate
activities.

## Why this lands

- "How do you debug what an agent did last Tuesday?" has no good answer in the ecosystem. Observability vendors show you dashboards; nobody gives you *replay*.
- Record/replay is a classic systems discipline (think rr, VCR cassettes, event sourcing) applied to a new domain.
- It completes a story no single repo can tell: the runtime produces behavior → blackbox captures it → the eval harness enforces it. Three repos that reference each other like a real platform.

## Design

### The trace format (`*.trace.jsonl`) — the real product

Append-only JSONL, one event per line, schema versioned (`docs/TRACE-FORMAT.md`
plus a JSON Schema in `schemas/`). Event types:

```jsonl
{"v":"0.1","type":"session_start","sessionId":"s1","at":"...","runtime":{"app":"spring-ai-agent-starter","model":"claude-sonnet-5"}}
{"type":"user_message","turn":1,"text":"What is 973 * 481?"}
{"type":"llm_request","turn":1,"seq":1,"messagesDigest":"sha256:...","toolsOffered":["calculator","clock"]}
{"type":"llm_response","turn":1,"seq":1,"stopReason":"tool_use","toolCalls":[{"id":"tu_1","name":"calculator","input":{"expression":"973 * 481"}}],"usage":{"in":412,"out":31},"millis":1840}
{"type":"tool_call","turn":1,"toolUseId":"tu_1","name":"calculator","input":{...}}
{"type":"tool_result","turn":1,"toolUseId":"tu_1","result":"468013","millis":2,"error":false}
{"type":"llm_response","turn":1,"seq":2,"stopReason":"end_turn","text":"973 × 481 = 468,013.","usage":{"in":455,"out":18},"millis":1210}
{"type":"assistant_message","turn":1,"text":"973 × 481 = 468,013."}
{"type":"error","turn":2,"where":"llm","message":"..."}          // failures are first-class events
{"type":"session_end","at":"..."}
```

Design rules to hold:

- **Failures are events, not gaps.** An error mid-turn is recorded exactly where it happened — the whole point is debugging the bad runs.
- **Full request payloads are NOT stored by default** — `messagesDigest` (hash) keeps traces small and privacy-sane while still letting replay detect divergence. `--capture-full` opts into complete payloads for deep debugging. This default-redacted stance is a README selling point, not a limitation.
- **Redaction on write:** configurable regex scrubbers (API keys, emails, card-like numbers) run before an event touches disk. Scrubbed spans are marked `"redacted":true`, never silently altered.
- One file per session, named `{sessionId}-{startedAt}.trace.jsonl`, in a configurable trace dir. Rotation/retention is a size cap on the dir (delete-oldest), documented.

### Recording: zero changes to the target

`blackbox-spring` is a Spring Boot auto-configuration that wraps the starter's
own abstractions via `BeanPostProcessor`: `LlmClient` and `ToolRegistry` get
decorated with recording proxies. **No code changes in spring-ai-agent-starter**
— add the dependency, get traces. That "instrument by decorating the seam"
design is the architectural insight to write up in DESIGN.md: the starter's
interfaces were the contract all along.

### Replay: deterministic, safe, judgmental

`blackbox replay trace.jsonl` boots a headless harness that **drives the real
agent loop** — not a scripted re-emission of the recorded steps. The replayer
feeds recorded model/tool outputs into the actual runtime and observes what the
current code does with them. This distinction is load-bearing: replay is only a
regression detector if it exercises the real loop (this was a review finding —
the first cut replayed a script and tested nothing).

- **`ReplayLlmClient`** feeds recorded `llm_response` events back in sequence — no API key, no network, no cost.
- **Safe by default: recorded tool results are returned instead of re-executing tools.** Replaying a session must never re-send an email. The default is *no real side-effects*; executing a real tool during replay requires an explicit opt-in flag, never the reverse (this was a review finding — the original default executed side-effects unless a tool was explicitly stubbed). Safety is non-negotiable and gets its own test.
- The replayer is **judgmental**: at each step it compares what the *current* agent code did (which tool it called, with what input) against what the trace recorded. A mismatch is a **divergence report** — "at turn 3, live code called `clock`, trace recorded `calculator`" — which is exactly how you catch a behavior regression after a refactor. Exit 0 = faithful, exit 1 = diverged (CI-compatible, same philosophy as mcp-pact).
- `--interactive` steps turn-by-turn (time-travel debugging in the terminal).

### Diff: `blackbox diff a.trace.jsonl b.trace.jsonl`

Aligns two traces of the same conversation turn-by-turn and reports per turn:
answer text similarity, tool-call sequence differences, token deltas, latency
deltas. Output as a human table and `--json`. This is the "compare prompt v1
vs v2 on the same input" demo — and pairs with agent-operator (canary produced
trace A, main produced trace B).

### Eval export: the killer command

```bash
blackbox export-eval trace.jsonl --turn 3 --out cases/incident-1234.yaml
```

Generates an eval case: the turn's user message as the prompt, `tool_called`
assertions from the recorded trajectory, and a `judge` assertion stub (criteria
templated from the recorded answer, `min_score` left for a human to confirm).
The output is deliberately a *draft* — the README should say a human reviews the
generated case, because auto-generated oracles that nobody reads are how eval
suites rot.

## Modules

| Module | Contents | Deps |
|---|---|---|
| `blackbox-core` | trace model, JSONL reader/writer, redaction, diff engine, eval exporter | Jackson only |
| `blackbox-spring` | auto-config, recording decorators for `LlmClient`/`ToolRegistry` | Spring Boot, starter interfaces |
| `blackbox-cli` | `replay`, `diff`, `export-eval`, `stats` (shaded jar) | core |

## Build phases

### Phase 0 — Design doc (half day)
`docs/DESIGN.md`: trace format with full schema, the decorate-the-seam recording
argument, replay safety rules (safe-by-default: never re-execute side-effects
unless explicitly allowed), the "replay drives the real loop" requirement,
divergence semantics, export philosophy (drafts, not oracles), non-goals (no UI,
no OTel — that's agent-meter's job; note the boundary explicitly since the two
repos will be compared). Commit first, alone.

### Phase 1 — core: format + reader/writer + redaction
Schema, writer (append, fsync-on-event configurable), reader (streaming,
tolerant of a truncated final line — crashes happen mid-write, that's the
point of a black box), redaction engine. Table-driven tests including the
truncated-tail case and redaction marking.

### Phase 2 — recording
`blackbox-spring` auto-config + decorators; integration test boots the actual
starter with a fake LlmClient and asserts the produced trace matches the
golden file byte-for-byte (modulo timestamps — normalize them).

### Phase 3 — replay + divergence
`ReplayLlmClient`, tool-result stubbing, replay driving the real agent loop,
divergence detection with exit codes, `--interactive`. e2e: record a session,
replay it green; then change a tool's behavior and replay red with a precise
divergence report. The safety test: a replay of a session containing
`send_email` must provably not invoke the real transport with the default flags.

### Phase 4 — diff + export-eval + README
Diff engine + CLI, eval exporter emitting valid agent-evals YAML (validate by
actually running agent-evals against it in CI — the repos test each other),
README with a 30-second GIF: incident trace → `export-eval` → failing eval →
fix → green.

## Guardrails
No employer names anywhere; the forbidden-string grep must return 0 pre-push;
HTTPS to the intended account only; design doc first; phases as separate PRs;
clean room.

## Acceptance checklist
- [ ] DESIGN.md + trace schema committed before code
- [ ] Starter records with zero code changes (dependency only)
- [ ] Replay drives the real agent loop, not a scripted re-emission
- [ ] Replay: green on faithful, red with precise divergence on drift, safe-by-default (provably no side-effects)
- [ ] Truncated-trace tolerance tested
- [ ] `export-eval` output validated by running agent-evals in CI
- [ ] README GIF: incident → eval case → green
- [ ] Public at github.com/hhagenbuch/agent-blackbox, forbidden-string grep clean
