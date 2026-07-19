# Runbook 7 — `agent-meter`: FinOps for agents

> **Produced:** `agent-meter`, OpenTelemetry-native cost attribution for LLM agents (per session, tool, prompt version, feature) plus budgets that degrade before they deny. **Repo:** https://github.com/hhagenbuch/agent-meter.
> **What the review cycle caught:** the README overclaimed scope — it implied provider-agnostic, real-agent instrumentation when the demo was synthetic and the metering seam was provider-specific. Corrected to state the true scope honestly (PRs #5, #7).

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `agent-meter` — OpenTelemetry-native token/cost attribution for LLM agents (per session, tool, prompt version, feature), with enforceable budgets that *degrade* before they *deny*.
**Builds on:** the agent runtime as the instrumented target. Pairs with `agent-blackbox` (that repo answers "what happened"; this one answers "what did it cost") — state the boundary in both READMEs.

## The pitch (top of README)

> Every company that shipped agents last year is now staring at the invoice and
> cannot answer the only question that matters: **which feature is spending
> this?** `agent-meter` instruments your agent with OpenTelemetry — standard
> `gen_ai.*` semantic conventions, so it lands in the Grafana you already have —
> and attributes every token and every cent to a session, a tool, a prompt
> version, and a feature. Then it enforces budgets the way a good SRE would:
> warn, degrade to a cheaper model, and only then block.

The philosophy line: **cost is a reliability dimension.** You'd never ship a
service without latency metrics; agents ship without cost metrics every day.

**Be precise about scope in the README** (this was a review finding): the
metering seam is the starter's `LlmClient` and the current cost/usage extraction
is provider-specific (Anthropic Messages `usage`); the shipped demo drives a
synthetic load, not a live production agent. State that plainly. The OTel
semconv output is portable; the extraction adapter is one provider today with
others as roadmap. Don't imply provider-agnostic or real-agent coverage the code
doesn't yet have.

## Why this lands

- The pain is universal, current, and visible to whoever holds the budget — this is the repo a VP forwards to their platform team.
- Building on the official OTel **GenAI semantic conventions** (`gen_ai.usage.input_tokens`, `gen_ai.request.model`, …) instead of inventing a format means the data works in Grafana/Datadog/Honeycomb on day one.
- Budget-enforcement-by-degradation reuses castaway's signature idea (explicit graceful degradation) in a different domain.

## Design

### Attribution model

Every LLM call span carries:

| Attribute | Source |
|---|---|
| `gen_ai.request.model` / `gen_ai.response.model` | client |
| `gen_ai.usage.input_tokens` / `output_tokens` | API response `usage` (provider-specific extraction) |
| `agent.session_id` | conversation |
| `agent.tool` | when the call resulted from a tool loop step |
| `agent.prompt_version` | from config/env (agent-operator sets this label on deployments — cross-repo synergy) |
| `agent.feature` | caller-supplied tag on the chat request (`ChatRequest.feature`), the FinOps unit |
| `agent.cost_usd` | computed by the cost engine (see below) |

Metrics (with exemplars linking back to spans): `agent.tokens` counter
(direction label in/out), `agent.cost_usd` counter, `agent.turn.duration`
histogram — all dimensioned by model, feature, prompt_version.

### The cost engine (`meter-core`) — pure and versioned

- **Price table as data, not code:** `prices.yaml` shipping in the jar, override via file/env. Entries carry `effective_from` dates and per-MTok input/output (and cached-input) rates; the engine picks the rate effective at call time.
- **Staleness is loud:** if the bundled table is older than 90 days at startup, log a prominent warning and set `agent.cost_estimated=true` on spans. Prices change; silently-wrong cost data is worse than none.
- **Unknown model:** cost recorded as *unknown* (attribute absent + warning metric incremented), never `0.00`. Zero is a lie that hides exactly the spend you most need to see.
- Pure functions, table-driven tests (rate boundaries around `effective_from`, cached-token pricing, unknown models).

### Budget enforcement (`meter-core` policy + `meter-spring` decorator)

Budgets are declarative:

```yaml
meter:
  budgets:
    - scope: feature:support-chat        # or session:*, or global
      limit_usd: 5.00
      window: daily
      on_breach: degrade                 # warn | degrade | block
      degrade_to: claude-haiku-4-5       # cheaper model for the remainder of the window
```

- Enforcement lives in an `LlmClient` decorator (same seam blackbox uses — the starter's interfaces keep paying rent). `warn` emits an event + metric; `degrade` swaps the model on the request and tags spans `agent.budget_degraded=true` (the user-visible provenance idea from castaway, applied to cost); `block` fails fast with a clear message.
- Windows tracked in-memory with a pluggable store interface (Redis impl as roadmap) — document the single-instance limitation honestly.
- Order of decorators matters (meter outside blackbox? inside?) — pick, test, and document it in DESIGN.md; reviewers will ask.

### Streaming and edge cases (design these in, don't discover them)

- Streamed responses report usage in the terminal event — the span stays open until then; a stream that dies mid-flight records tokens-so-far with `agent.incomplete=true`.
- Retries: each HTTP attempt is a child span; cost counts **every** attempt (you pay for retries — that visibility is a feature; the README should show a retry-storm screenshot as a selling point).
- Prompt content never goes in attributes by default (`gen_ai` semconv marks content as opt-in) — privacy stance consistent with blackbox.

### The demo stack (`demo/`)

`docker-compose up`: otel-collector → Prometheus + Tempo → Grafana with a
**shipped dashboard JSON** — cost by feature, tokens by model, budget burn-down,
top-10 expensive sessions (click through exemplar → trace). Plus a `load.sh`
that drives the starter through mixed features so the dashboard is alive in 60
seconds. Label the demo load as synthetic in the README — it exercises the
metering path end-to-end, but it is a load generator, not a production agent.
The dashboard screenshot is the README hero image.

## Modules

| Module | Contents | Deps |
|---|---|---|
| `meter-core` | cost engine, price table, budget policy — pure | Jackson, OTel API only |
| `meter-spring` | auto-config, `LlmClient` metering + budget decorators, `ChatRequest.feature` propagation | Spring Boot, OTel SDK |
| `demo/` | compose stack, Grafana dashboard JSON, load script | — |

## Build phases

### Phase 0 — Design doc (half day)
`docs/DESIGN.md`: attribution table, semconv adherence (cite the `gen_ai`
spec version), cost-engine rules (staleness, unknown-model, retries-count),
budget state machine (warn → degrade → block), decorator ordering, and an
explicit **scope statement** (provider-specific extraction today, synthetic
demo load, OTel output portable), non-goals (no UI of our own, no multi-instance
budget store in MVP, no proxy/gateway mode — decorator only). Commit first, alone.

### Phase 1 — cost engine
`prices.yaml` schema + engine + budget policy as pure code. The test suite is
the spec: effective-date boundaries, cached pricing, unknown model, window
rollover, degrade-then-recover at window reset.

### Phase 2 — OTel instrumentation
`meter-spring` decorator emitting spans/metrics per the attribution table;
usage extraction from the starter's Anthropic responses (and the streaming
terminal event); retry child spans. Integration test with OTel's
`InMemorySpanExporter` asserting exact attributes — no collector needed in CI.

### Phase 3 — budget enforcement
Decorator wiring for warn/degrade/block; degrade proven end-to-end with a fake
client (request model actually swapped, span tagged); `block` message quality
checked by a test (it will be read by an annoyed developer at 2am).

### Phase 4 — demo stack + README
Compose stack, dashboard JSON, load script, hero screenshot, README with the
"cost is a reliability dimension" framing, budget YAML cookbook, an honest scope
statement, and explicit boundary notes versus agent-blackbox and versus vendor
observability suites (standard OTel = no lock-in is the differentiator).

## Guardrails
No employer names anywhere; the forbidden-string grep must return 0 pre-push;
HTTPS to the intended account only; design doc first; phases as separate PRs;
clean room. Price table values come from public pricing pages — cite the
retrieval date in a comment.

## Acceptance checklist
- [ ] DESIGN.md committed before code, semconv version cited, scope stated honestly
- [ ] Cost engine: table-driven tests incl. staleness, unknown-model, retry accounting
- [ ] Spans/metrics verified attribute-exact via InMemorySpanExporter in CI
- [ ] Degrade path proven end-to-end (model actually swapped, tagged, recovers at window reset)
- [ ] README does not overclaim: provider-specific extraction and synthetic demo load stated plainly
- [ ] `docker-compose up` → live Grafana dashboard in under 2 minutes; screenshot in README
- [ ] Public at github.com/hhagenbuch/agent-meter, forbidden-string grep clean
