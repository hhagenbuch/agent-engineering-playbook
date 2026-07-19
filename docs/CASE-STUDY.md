# Case study: building `castaway` under review

One build, traced end to end — from a design document to a measured benchmark
whose result validated the architecture. `castaway` is the richest example in
the set because its correctness claims are distributed-systems claims, the kind
that are easy to assert and hard to earn, so the review cycle had real work to do.

**Repo:** https://github.com/hhagenbuch/castaway ·
**Design RFC:** [`docs/DESIGN.md`](https://github.com/hhagenbuch/castaway/blob/main/docs/DESIGN.md)

## What it is

An agent runtime for intermittent connectivity — ships on satellite links,
mines, aircraft, rural clinics. Instead of failing when the API is unreachable,
it degrades explicitly: a `LinkMonitor` tracks `ONLINE / DEGRADED / OFFLINE`
with hysteresis, a `ModelRouter` falls back from the cloud model to a local one
per request, a `CapabilityGate` lets tools declare what connectivity they need
and negotiates scope when degraded, a durable `Outbox` queues side effects and
revalidates them on reconnect, and an append-only `MemoryLog` ships its log to
shore when the link returns. Six components, each a distributed-systems problem
pointed at a new domain.

That description is exactly the kind of thing an agent will happily write and
not actually deliver. The method is what made the difference between the claim
and the system.

## The timeline

Each phase was its own pull request against a design that was reviewed first.
The two review-cycle rows are where approval was withheld until a real finding
was fixed.

| # | PR | What it delivered | What review caught |
|---|----|-------------------|--------------------|
| — | design RFC | The six-component architecture, the link state machine, the at-most-once outbox premise | Design signed off *before* code — the architecture was the thing under review |
| 1 | [#1](https://github.com/hhagenbuch/castaway/pull/1) Phase 1: skeleton, LinkMonitor, ModelRouter | Compiling skeleton; link state machine; cloud/local routing | — (skeleton phase) |
| 2 | [#2](https://github.com/hhagenbuch/castaway/pull/2) Phase 1 hardening | Call-level fallback (route mid-call, not just at startup); self-healing probe stream | Startup-only routing wouldn't survive a link drop *during* a call — hardened to fall back per call |
| 3 | [#3](https://github.com/hhagenbuch/castaway/pull/3) Phase 2 | `CapabilityGate`; revalidating SQLite `Outbox`; deferred actions | — |
| 4 | [#4](https://github.com/hhagenbuch/castaway/pull/4) Phase 3 | `MemoryLog` event-log sync; chaos harness (Toxiproxy); evals under partition | — |
| 5 | [#5](https://github.com/hhagenbuch/castaway/pull/5) Phase 4 | Verified offline demo; local-model benchmark; written design decisions | — |
| 6 | **[#6](https://github.com/hhagenbuch/castaway/pull/6) Review cycle 1: orphaned-outbox sweep** | `ORPHANED` terminal state; startup sweep; documented at-most-once | **A crash between `REVALIDATED` and `EXECUTED` stranded an entry the reconciler never looked at again — invisible forever** |
| 7 | **[#7](https://github.com/hhagenbuch/castaway/pull/7) Review cycle 2: measured benchmark** | Real `llama3.1:8b` and corrected `qwen3:8b` numbers from actual runs | **The benchmark table had a placeholder row — replaced with real measured data, and the finding changed the recommendation** |
| 9 | [#9](https://github.com/hhagenbuch/castaway/pull/9) | Spring Boot 3.5.12 + Java 25 (LTS) | Platform currency |

## Review cycle 1: the invisible crash window

The outbox advances an entry to `REVALIDATED` *before* the side effect fires —
deliberately, because that ordering makes execution at-most-once (better to drop
an irreversible action than to double-send it). Review followed the state machine
one step further than the happy path and found the hole: if the process crashed
after `REVALIDATED` but before `EXECUTED`, or the executor threw on an unknown
action type, the entry was stuck. The reconciler only reconsidered `QUEUED`
entries, so a stranded one was never retried and never surfaced — visible only to
someone who thought to read `GET /api/outbox`.

The finding was written down plainly and the merge was withheld. The fix
([#6](https://github.com/hhagenbuch/castaway/pull/6)) added an `ORPHANED`
terminal state (not retried — that could double-send — but *surfaced* for
review), made the reconciler mark execution failures inline instead of leaving
them stuck, and added a startup sweep so a crash-stranded entry becomes visible
on the next boot. It also wrote the at-most-once guarantee into the design docs,
because an undocumented safety trade-off is a trap for the next reader. Two tests
were added to hold the behavior.

This is the whole review discipline in one PR: a finding traced past the happy
path, a fix that preserves the intended semantics rather than papering over them,
and the trade-off documented so it survives.

## Review cycle 2: the benchmark that validated the architecture

Phase 4 shipped with a benchmark table that had a documented gap — a placeholder
row for a second local model. A placeholder in a table that exists to justify a
recommendation is a claim with no evidence, so it was sent back for real numbers.

The measured run ([#7](https://github.com/hhagenbuch/castaway/pull/7)) is the
most important result in the project, and not because the numbers were good. They
were revealing. Running the actual offline flow against `llama3.1:8b`:

- it **over-declined** a plain knowledge question on a cold call,
- it fired **spurious tool calls** — invoking `calculator` and `clock` on a
  definition question,
- it narrated the queued email by referencing a **tool that does not exist**.

And yet: it queued the email correctly — one entry, safe — because the durable
outbox, the idempotency key, and the capability gate do not depend on the model
behaving well. The corrected `qwen3:8b` numbers told the cleaner story (no
spurious tool calls, honest narration), which is why it became the offline
default. But the `llama3.1:8b` row is the one that mattered: **a poorly-behaved
model could not produce an unsafe outcome, because the safety was in the
architecture, not the model's discipline.**

That is the thesis of the whole system, and the benchmark is what turned it from
an assertion in a design doc into a demonstrated property. It could only do that
because review refused to accept the placeholder — the honesty gate and the
architecture validation were the same act.

## What the case study shows

- **Design-doc-first** meant the six-component architecture was critiqued as an
  idea, before three thousand lines were written against it.
- **Phasing** kept each review tractable and forced an architecture commitment
  early, at the skeleton, when changing it was cheap.
- **Adversarial review** traced the outbox state machine one step past the happy
  path and found a data-loss window the tests did not.
- **Honesty over silent degradation** turned a placeholder benchmark row into the
  result that validated the design — because the same gate that rejected the
  placeholder demanded the real run.

None of these required a better agent. They required treating the agent's output
the way you would treat a strong engineer's: with a spec up front, a review that
assumes nothing, and a bar that does not move.
