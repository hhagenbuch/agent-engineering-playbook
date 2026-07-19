# The Agent Engineering Playbook

### How to run an engineering org of AI agents: specs, review gates, and quality bars

An AI coding agent can write a plausible module in a minute. It can also
overclaim in a README, invert a safety default, and quietly drop the one
integration it promised — in the same minute, with the same confidence. The
gap between those two outcomes is not model quality. It is management.

This repository is an argument, backed by receipts, for a single claim:

> **AI agents are a workforce. The discipline that makes a workforce produce
> production-grade work is the same whether the workers are human or not:
> written specifications, scoped autonomy, adversarial review, explicit
> acceptance criteria, and quality gates that block on failure.**

The evidence is public. Over a short, intense stretch I directed AI agents to
build seven non-trivial systems — an agent runtime, an evaluation harness, a
contract-testing tool for the Model Context Protocol, a network-resilient
agent, a Kubernetes operator, a session flight-recorder, and a cost meter. I
did not write most of the code. I wrote the specs, set the guardrails, ran the
review, and refused to merge work that could not defend itself. Every repository
carries the pull-request history that shows the method being followed —
including the parts where review caught something real and sent it back.

The `runbooks/` here are those specs, sanitized. `docs/CASE-STUDY.md` traces one
build end to end. `docs/TEMPLATE.md` is a blank you can copy. What follows is the
method.

---

## The operating model: a runbook is a spec, not a prompt

The unit of work is not a chat message. It is a **runbook** — a written
specification handed to an agent the way you would hand a scoped project to a
capable engineer you trust but do not micromanage. Every runbook has the same
four parts, and the parts are load-bearing.

**Context.** What exists, what this connects to, what "done" looks like in one
paragraph. An agent with no context invents context, and invented context is
where overconfident wrong work comes from. The context section is the cheapest
insurance you can buy: a few sentences that keep the agent inside reality.

**Guardrails.** The constraints that are not up for negotiation — architectural
standards, the layering that must hold, the things that must never happen. This
is where you encode hard-won taste so you do not have to re-explain it every
turn. A concrete example that recurs across these builds: a **forbidden-string
grep that must return zero before every commit** — a mechanical gate that fails
the build if a banned token (an internal codename, a leaked secret pattern)
appears anywhere. It is trivial to write and it never gets tired. Guardrails are
the difference between "autonomy" and "abdication."

**Phases.** The build is broken into ordered phases, each shippable and each its
own pull request. Phase 1 is a skeleton that compiles. Phase 2 adds the risky
core. Later phases add the integration and the polish. Phasing does two things:
it keeps any single review tractable, and it makes the agent commit to an
architecture *before* it has written three thousand lines against a wrong one.
The design document comes first, and it is reviewed first — **design-doc-first**
is the cheapest place to catch a bad idea.

**Acceptance.** A checklist of what must be true for the phase to be done, written
*before* the work starts. Not "it looks good" — "`mvn verify` is green," "an
execute() failure orphans the entry and a second pass does not retry it," "the
demo renders from a fresh clone." Acceptance criteria written in advance are the
only defense against an agent (or a person) grading its own work generously
after the fact.

Scoped autonomy is the whole game. Too tight and you are just a slow typist with
extra steps. Too loose and you get confident nonsense. The runbook is where you
set the scope: here is the context, here are the lines you may not cross, build
it in these phases, and it is not done until these things are true.

---

## The review discipline: nothing merges unexamined

Every feature in these repositories went through an adversarial review before it
merged. Not a rubber stamp — a review whose job was to *find the thing that is
wrong*, write it down, and withhold the merge until it was fixed and the fix was
verified. The receipts are the pull requests whose descriptions literally answer
a review finding:

- **[agent-blackbox #4 — "Wire replay to the real agent; make replay safe by
  default"](https://github.com/hhagenbuch/agent-blackbox/pull/4).** Review found
  that "replay" did not actually re-run the agent (it replayed recorded tool
  calls only), and — worse — that its safety default was inverted: it *executed*
  real side effects unless you remembered to stub them. Both were sent back. The
  replay was rewritten to drive the real agent loop, and the default was flipped
  to never execute unless you opt in with `--execute`.

- **[castaway #6 — "Sweep orphaned outbox entries; document at-most-once
  execution"](https://github.com/hhagenbuch/castaway/pull/6).** Review traced a
  crash window: an entry could reach a `REVALIDATED` state, fail before
  execution, and then be invisible forever because the reconciler only looked at
  `QUEUED`. The fix added an `ORPHANED` terminal state, a startup sweep, and a
  written statement of the at-most-once guarantee it was trading for.

- **[agent-meter #5 — "Docs: correct scope — provider-agnostic seam, synthetic
  demo, aggregate retries"](https://github.com/hhagenbuch/agent-meter/pull/5).**
  Review caught the README claiming more than the code delivered — implying a
  real-agent, provider-agnostic integration when the demo was synthetic and the
  instrumentation seam was provider-specific. The fix was not to the code. It was
  to the claims.

- **[mcp-pact #1 — "Close silent-pass holes in the diff; harden matchers, exit
  codes, capabilities"](https://github.com/hhagenbuch/mcp-pact/pull/1).** Review
  found paths where a contract check could *pass silently* on input it should
  have flagged — the worst failure mode for a testing tool, because it launders
  false confidence. Every hole was closed and given a test.

The pattern is always the same: a finding is stated plainly, the fix is made,
the fix is verified, and only then does it merge. A review that produces no
written findings and blocks nothing is not a review. It is a signature.

---

## The doctrines that emerged

Run enough of these cycles and a small number of principles keep earning their
place. They are not style preferences. Each one is a rule that caught a real
defect.

**Honesty over silent degradation.** When a system cannot do the thing, it must
say so — loudly, at the seam, with an exit code or an error — never paper over
it and return something plausible. agent-evals was made to [fail loudly on
non-retryable judge errors instead of parsing garbage](https://github.com/hhagenbuch/agent-evals/pull/3);
agent-meter's budget policy [degrades before it denies](https://github.com/hhagenbuch/agent-meter/pull/3),
and says which it did. A tool that hides its own degradation is worse than one
that breaks, because it spends your trust without telling you.

**Pure core, effect shell.** The decision logic is a pure function; the I/O
lives at the edges. agent-operator's promotion logic is a pure
`decide(phase, outcome, deadlineExceeded)` with the Kubernetes calls pushed to
the shell; mcp-pact's diff is a pure comparison over two captured contracts.
Pure cores are the parts you can actually test exhaustively, and agents write
correct pure functions far more reliably than they wire correct side effects.

**Design-doc-first.** No phase starts without a design the reviewer signed off
on. The cheapest bug to fix is the one still written in prose.

**Exit-code contracts.** Anything that runs in CI has a defined contract: `0`
means the invariant holds, non-zero means it is broken, and the meaning of each
code is documented. mcp-pact and agent-blackbox both treat their exit codes as
API. This is what lets a quality gate actually *gate*.

**We either understood a change or we flagged it.** When a comparison could not
be confidently classified as safe, it was not passed — it was surfaced. mcp-pact
ships a whole **WARN** tier for exactly this: a tool-description change is not
"breaking" and not "compatible," it is "a human should look." The refusal to
guess is a feature.

---

## What transfers to managing people, and what does not

Most of it transfers, which is the uncomfortable and useful part.

The specification discipline is identical. A vague runbook produces vague work
from an agent for the same reason a vague ticket produces the wrong feature from
a person: ambiguity gets resolved in the direction of least effort, and you do
not control which direction that is unless you wrote it down. Scoped autonomy is
identical — good engineers and good agents both do their best work when the
boundaries are explicit and the middle is theirs. Review is identical: the value
is in the written finding and the withheld approval, not the reading. Acceptance
criteria are identical, and for the same reason — nobody grades their own
homework honestly.

What does not transfer is as important to name. An agent has no growth arc you
are responsible for; you are not building a career, you are extracting a task,
and the humane parts of management — mentorship, trust built over time, the
benefit of the doubt — are category errors when the worker is a model. An agent
will not learn from Tuesday's review by Wednesday unless you put Tuesday's lesson
back into the spec; institutional memory lives in your runbooks and guardrails,
not in the worker. And an agent's confidence is uncorrelated with its
correctness in a way no experienced human's is, which means you cannot use the
usual human tells. You must verify. The trust you extend a senior engineer is
earned and calibrated; the "trust" you extend an agent is a policy you enforce
with gates, because the agent will state a wrong answer with exactly the same
fluency as a right one.

So: manage the work like a person's, and audit the output like a machine's.

---

## Failure modes observed

These are not hypotheticals. Each was caught in these builds, by review, and
fixed in the open.

- **Overclaiming in the README.** The most common failure. Agents write
  documentation that describes the system they were asked to build, not the one
  they built — provider-agnostic when it was provider-specific, "real" when the
  demo was synthetic. Caught and corrected in
  [agent-meter #5](https://github.com/hhagenbuch/agent-meter/pull/5) and
  [#7](https://github.com/hhagenbuch/agent-meter/pull/7). Mitigation: treat the
  README as a claim under audit, and make acceptance criteria demand that the
  demo run from a fresh clone.

- **Safety defaults inverted.** A replay tool that executed real side effects
  unless told not to — the dangerous default presented as the convenient one.
  Caught in [agent-blackbox #4](https://github.com/hhagenbuch/agent-blackbox/pull/4).
  Mitigation: name the safe default in the guardrails, explicitly, before the
  agent chooses one for you.

- **Integration promises silently dropped.** A phase claims an end-to-end
  integration, and the seam quietly turns out to be a stub or a structural check
  rather than the real cross-system run. Caught by demanding the loop actually
  close — the cross-repo eval that runs a real published image, the live smoke
  that hits the real API. Mitigation: acceptance criteria for integration must be
  "it ran," not "it would run."

- **Plausible-but-wrong parsing.** Code that handles the response shape the agent
  imagined rather than the one the API returns — for example reading the first
  content block as text when the model can emit a reasoning block first. Caught
  in [agent-evals #10](https://github.com/hhagenbuch/agent-evals/pull/10), and
  then the fix was checked against every other place that parsed the same shape,
  not just the one that failed.

The through-line: none of these are caught by a better prompt. They are caught by
a review that assumes the work is wrong until it proves otherwise, and by
acceptance criteria written before the work so the bar cannot move to meet it.

---

## Read the receipts

- [`runbooks/`](runbooks/) — the seven specifications, sanitized, each headed
  with what it produced and what its review cycle caught.
- [`docs/CASE-STUDY.md`](docs/CASE-STUDY.md) — one build (castaway) traced from
  design document through phase PRs, two review cycles, and a measured benchmark
  whose finding validated the architecture.
- [`docs/TEMPLATE.md`](docs/TEMPLATE.md) — a blank runbook to copy.

The method is not complicated. It is just unfashionable to apply it to a worker
that types this fast. Do it anyway. The speed is the reason the discipline
matters more, not less.
