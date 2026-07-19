# Runbook 3 — `castaway`: an agent runtime for disconnected environments

> **Produced:** `castaway`, a network-resilient agent runtime — local-model fallback when the link drops, a durable outbox for deferred side-effects, and a capability gate that degrades the toolset explicitly. **Repo:** https://github.com/hhagenbuch/castaway.
> **What the review cycle caught:** outbox entries could strand in a `REVALIDATED` state after a crash and never reconcile — fixed with an `ORPHANED` terminal state, a startup sweep, and documented at-most-once semantics (PR #6); a placeholder benchmark row in the README was replaced with real measured data (PR #7).

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `castaway` — design doc first, then a working MVP that demonstrably survives a network partition mid-conversation.
**Builds on:** the agent runtime and eval harness from Runbook 1 (castaway reuses the starter's abstractions and is evaluated by the eval harness).

## The pitch (put this at the top of the README)

> Every agent framework assumes the API is always there. The real world has
> ships on satellite links, mines, aircraft, rural clinics, and factory floors
> where connectivity is intermittent, expensive, and slow. `castaway` is an
> agent runtime built for that world: it degrades explicitly instead of
> failing, routes to a local model when the link drops, queues side-effects
> for reconciliation, and syncs memory when the connection returns.

Positioning rule: the repo never names an employer or a specific deployment. Intermittent-connectivity environments appear only as a list of examples.

## Why this is a category of one

- Edge-AI people don't build agents; agent people assume the cloud. The intersection is nearly empty.
- Every design problem here is a classic distributed-systems problem (outbox, idempotency, conflict resolution, partition tolerance) applied to a new domain — it showcases systems thinking, not prompt engineering.
- It has a spectacular live demo: pull the plug mid-conversation and the agent keeps going.

## Architecture

```
                       ┌─────────────────────────────────────────┐
                       │              castaway runtime           │
User ──► Chat API ──►  │  AgentLoop (from spring-ai-agent-starter)│
                       │      │                                  │
                       │  ModelRouter ──► CloudLlmClient (Anthropic)
                       │      │       └─► LocalLlmClient (Ollama)│
                       │      ▲                                  │
                       │  LinkMonitor (ONLINE/DEGRADED/OFFLINE)  │
                       │      │                                  │
                       │  CapabilityGate ──► ToolRegistry        │
                       │      │         (tools declare link needs)│
                       │  Outbox ──► reconciler (on reconnect)   │
                       │  MemoryLog ──► sync (on reconnect)      │
                       └─────────────────────────────────────────┘
```

Six components, each a topic in itself:

1. **LinkMonitor** — a state machine (`ONLINE`, `DEGRADED`, `OFFLINE`) fed by
   active probes (latency/loss to the cloud endpoint) with hysteresis so it
   doesn't flap. Emits transitions as events; everything else subscribes.
   `DEGRADED` matters: satellite links aren't down, they're 700ms RTT and 5%
   loss — the router should prefer local for long generations but still use
   cloud for hard reasoning.

2. **ModelRouter** — implements the starter's existing `LlmClient` interface
   (this is the payoff of that abstraction) and delegates per-request:
   `ONLINE` → cloud; `DEGRADED` → policy choice (cost/latency budget);
   `OFFLINE` → local model via Ollama. Every response is tagged with
   provenance (`answered-by: local-qwen3-8b, link: OFFLINE`) surfaced to the
   user — honesty about degradation is a design principle, not a footnote.

3. **CapabilityGate** — tools declare their link requirement:
   `OFFLINE_CAPABLE` (calculator, local search), `DEFERRABLE` (send email —
   can be queued), `ONLINE_ONLY` (live pricing). Offline, the gate rewrites
   the toolset the model sees and injects a system-prompt notice: "you are
   offline; you may draft but not execute deferrable actions; say so."
   The agent's *knowledge of its own degradation* is the novel UX.

4. **Outbox** — deferrable side-effects go to a durable queue (SQLite via
   JDBC — embedded, no server, survives restart) with an idempotency key,
   the conversational context that produced them, and a TTL. On reconnect the
   reconciler replays them — but first **revalidates**: an action queued 6
   hours ago may no longer make sense ("the meeting it was rescheduling has
   passed"). Revalidation = a cloud-model check of action-against-current-state
   before execution. Stale actions surface to the user instead of firing. This
   revalidation step is the most original idea in the project — do not cut it.

5. **MemoryLog** — conversation memory as an append-only event log (not
   mutable state), so sync between two instances is log shipping + merge, not
   diffing. Conflicts (the user talked to a second agent while this one was
   offline) are resolved last-writer-wins per session with both branches
   retained; a cloud-model summarization pass folds the divergent branch back
   in as context ("while you were offline, you also asked X elsewhere").

6. **Chaos harness** — Toxiproxy between castaway and the cloud API, driven
   by scenario scripts: `partition.sh`, `satellite.sh` (700ms + 5% loss +
   200kbps), `flap.sh`. Network conditions are test fixtures, not accidents.

## Local model reality check (bake into the design, don't discover it late)

Small local models are much worse at tool calling. Mitigations, in order:
constrain offline tool calls with strict JSON schemas and retry-on-invalid;
shrink the offline toolset to 2–3 tools (CapabilityGate already does this);
prefer models with decent function-calling at 4–8B (test Qwen3, Llama 3.x
class models via Ollama; pick empirically with the eval suite, and record the
comparison in the README as a measured benchmark table — never a placeholder).
If tool calling is still unreliable offline, fall back to "offline = Q&A +
drafting only, no tool execution" — an honest, defensible design decision.

## Build phases

### Phase 0 — Design doc first (half a day, huge signal)

Create the repo with `docs/DESIGN.md` before any code: problem, the six
components, the offline-action state machine (`PROPOSED → QUEUED → REVALIDATED
→ EXECUTED | STALE | REJECTED`), consistency model for MemoryLog, and explicit
non-goals (no multi-agent, no k8s yet, no fine-tuning). Commit it alone:
"Design: agent runtime for disconnected environments". A repo that opens with
an RFC reads senior. Reuse this runbook's content freely.

**Design the crash paths into the state machine now.** An entry mid-way through
reconciliation (e.g. `REVALIDATED` but not yet `EXECUTED`) must have a defined
fate after a process crash — otherwise it strands forever. Add an `ORPHANED`
terminal state and a startup sweep that reconciles or orphans any entry left in
a transient state, and document the delivery guarantee explicitly (at-most-once
for executed side-effects). This was a real review finding; get it into the
design rather than discovering it in production.

### Phase 1 — Skeleton + LinkMonitor + ModelRouter

- New repo. Same stack as the starter (Boot 3.3, Java 21, WebFlux). Either
  depend on `spring-ai-agent-starter` via JitPack or copy the ~6 core classes
  with attribution; copying is fine and keeps castaway self-contained — decide
  and note it in DESIGN.md.
- `LocalLlmClient` against Ollama's `/api/chat` (it speaks an
  OpenAI-compatible tools format; adapt to the internal `LlmResponse`).
- LinkMonitor with probe + hysteresis; expose state at `GET /api/link` and
  as an SSE stream (the dashboard/demo hooks onto this).
- Acceptance: with Ollama running and Wi-Fi off, `POST /api/chat` still
  answers, response tagged `link: OFFLINE`. Unit tests fake the monitor.

### Phase 2 — CapabilityGate + Outbox + reconciler

- Tool metadata (`LinkRequirement` enum on `AgentTool`), gate filtering,
  offline system-prompt injection.
- SQLite outbox, idempotency keys, `SendEmailTool` (fake transport) as the
  canonical deferrable action.
- Reconciler with the revalidation pass. Acceptance test (the demo script):
  offline → "email Bob asking to move the meeting to 3pm" → agent drafts,
  queues, says so → reconnect → revalidation approves → executes → user
  notified. Second test: queue an action, make it stale (TTL or contradicting
  fact), verify it surfaces instead of executing. Third test: crash between
  revalidation and execution, restart, confirm the startup sweep resolves the
  entry (executed once or orphaned — never stranded, never double-fired).

### Phase 3 — MemoryLog sync + chaos harness + evals

- Event-log memory, two-instance sync demo (two castaway processes, one behind
  Toxiproxy, one not), divergence + fold-back summarization.
- Toxiproxy scenario scripts under `chaos/`.
- **Evals under partition:** an eval dataset per network profile — the same
  golden cases run ONLINE and OFFLINE with different thresholds (offline answers
  may be weaker but must never pretend to be online, never claim an action
  executed when it was queued). Assert honesty with `judge` criteria:
  "acknowledges being offline / does not claim the email was sent". Running the
  eval harness against the runtime under simulated satellite conditions is the
  strongest evidence in the repo.

### Phase 4 — Demo + README (do not skip)

- Terminal or minimal web demo, recorded GIF: conversation → `chaos/partition.sh`
  → visible `OFFLINE` banner → agent keeps working, queues an email → restore
  → reconciliation messages. 30 seconds, autoplay in the README.
- README: pitch (above), architecture diagram, GIF, local-model benchmark
  table (measured, not placeholder), design-decisions section, link to
  DESIGN.md, roadmap (multi-node memory mesh, k8s operator integration).

## Guardrails

1. No employer names/domains anywhere in the repo; the forbidden-string grep
   must return 0 before every push; HTTPS to the intended account only.
2. "Ships" may appear as one example among several (aircraft, mines, clinics) —
   never as the framing story of the repo.
3. Don't gold-plate Phase 1–2 infrastructure; the differentiators are
   revalidation, honesty-under-degradation, and evals-under-partition. If
   time-boxed, cut the two-instance sync (Phase 3) before cutting those.
4. Commit cadence: each phase = multiple commits, design doc first.

## Prereqs

```bash
brew install ollama toxiproxy   # toxiproxy: server + cli
ollama pull qwen3:8b            # pick final model empirically in Phase 1
```

## Acceptance checklist

- [ ] DESIGN.md committed before any code
- [ ] Wi-Fi-off chat works via Ollama, provenance-tagged
- [ ] Queue → revalidate → execute/stale flow covered by tests and the demo
- [ ] Crash mid-reconcile resolves via startup sweep (at-most-once, no strands)
- [ ] Evals-under-partition datasets green in CI (deterministic tier keyless)
- [ ] README with GIF, measured benchmark table, no employer references
- [ ] Public at github.com/hhagenbuch/castaway
