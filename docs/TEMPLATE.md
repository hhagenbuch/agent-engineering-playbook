# Runbook template

Copy this file, fill every section, and hand it to the agent as the whole spec.
The section order is deliberate: context and guardrails come before the work so
the agent builds inside reality and inside your constraints. Acceptance comes
last and is written *now*, before the work, so the bar cannot move later to meet
whatever got built.

Delete the italic guidance as you fill each section.

---

## 1. Context

*One paragraph. What already exists, what this connects to, and what "done" looks
like at the end. If the agent would have to guess something to proceed, it goes
here instead. Name the repo(s), the language/stack, and the one-sentence purpose.*

- **Repo / location:**
- **Stack:**
- **Purpose (one sentence):**
- **What exists now:**
- **What this connects to:**

## 2. Guardrails

*The constraints that are not up for negotiation. Architectural standards, the
layering that must hold, the things that must never happen, the safe defaults you
are naming explicitly so the agent does not choose one for you. Keep these
short and absolute.*

- **Architecture / layering:**
- **Must never:** *(e.g. block the reactive thread; execute a side effect by
  default; log a secret)*
- **Safe defaults (state them — don't assume):**
- **Hard gate(s):** *(a mechanical check that fails the build — e.g. a
  forbidden-string grep that must return 0, a lint/format gate, a coverage floor)*

## 3. Design first

*No code until the design is written and reviewed. State what the agent must
produce as a design artifact before Phase 1, and that it waits for sign-off.*

- **Design deliverable:** *(e.g. `docs/DESIGN.md` — components, data model, the
  one hard trade-off and which way it's decided, exit-code/API contract)*
- [ ] Design reviewed and signed off before Phase 1 begins

## 4. Phases

*Break the build into ordered, individually-shippable phases — each its own pull
request. Phase 1 compiles. The risky core comes early (so a wrong architecture is
caught cheap). Integration and polish come later. For each phase: what it
delivers and how it will be reviewed.*

### Phase 1 — *(skeleton that compiles)*
- Delivers:
- PR:

### Phase 2 — *(the risky core)*
- Delivers:
- PR:

### Phase 3 — *(integration — the seam that must actually run, not a stub)*
- Delivers:
- PR:

### Phase 4 — *(demo, docs, polish)*
- Delivers:
- PR:

## 5. Acceptance criteria

*Written before the work. Concrete and checkable — "the build is green," "this
specific failure orphans the entry and a second pass does not retry it," "the
demo runs from a fresh clone" — never "it looks good." Integration criteria must
be "it ran," not "it would run."*

- [ ] Build/test gate green (name the command)
- [ ] *(behavioral criterion — a specific input produces a specific verified outcome)*
- [ ] *(integration criterion — the real cross-system path executed, not a stub)*
- [ ] Demo/quickstart works from a fresh clone
- [ ] README claims match what the code actually does (no overclaiming)
- [ ] Hard gate(s) from §2 return clean

## 6. Review expectations

*State the review contract so the agent knows the work will be adversarially
examined and that findings block the merge.*

- Every phase PR is reviewed to **find what is wrong**, not to approve.
- Findings are written down; the merge is withheld until each is fixed **and the
  fix is verified**.
- A change that cannot be confidently classified as safe is **flagged, not
  passed** (surface it; don't guess).
- The PR description should answer any review finding it resolves.
