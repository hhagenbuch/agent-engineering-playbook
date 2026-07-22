# Runbook 8 — `conductor`: the multi-agent layer

> **Produced:** `conductor`, session orchestration for Claude Code — a bus, work leases, and one command that spawns a briefed helper session. **Repo:** https://github.com/hhagenbuch/conductor.
> **What the review cycle caught:** the motivation claim was overstated ("leases prevent the incidents") when both incidents were git commands that bypass a Write/Edit hook — corrected to face it (a best-effort Bash git classifier *and* a precise rewrite: file conflicts → leases, git-level → worktree+PR discipline). A committed consent file would have opted in future contributors' local transcripts — made local-only and self-gitignored. And the dogfood surfaced that `assist` is intra-repo, not the cross-repo tool the runbook assumed.

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `conductor` — the layer that lets Claude Code sessions see each
other, message each other, prevent conflicting writes, and throw a second
session at a job in progress.
**Builds on:** every other repo. It reuses `agent-blackbox`'s redaction as a
library, contract-tests its own bus with `mcp-pact`, and cites `agent-medic`'s
Surgeon authority boundary. It is the layer *above* the rest of this playbook.

## The pitch (top of README)

> Sessions are workers; work needs a bus. Claude Code coordinates subagents
> *within* a session and nothing *across* sessions — no awareness, no
> messaging, no conflict prevention, no way to throw a second session at a job.
> conductor is that missing layer, built from three mechanics Claude Code
> already exposes: an MCP server for the bus, hooks for registration and
> enforcement, and transcripts (consented, redacted) for awareness.

The philosophy line: **the coordinator schedules, informs, and blocks — it
never edits, merges, or approves.** Same authority boundary as agent-medic's
Surgeon: system-wide visibility earns observation and veto, never a pen.

## Why this is the chapter the method was missing

The rest of this playbook is a method for running **one** agent through **one**
runbook: a spec, a design-doc gate, phased PRs, a review at each seam. It
sequences. It says nothing about what happens when **two** sessions touch the
same project at once — which is exactly how this portfolio's two worst
incidents happened: a session that branched from a stale local `main` and
squash-merged another session's unpushed commits, and a session that published
draft docs another was still writing. Two uncoordinated sessions, one project,
both times.

conductor is the layer that makes the method safe to parallelize:

- **Registry + `who_else`** — sessions register on start (a `SessionStart`
  hook) and are mutually visible; you can no longer work blind next to another
  session.
- **Leases + a PreToolUse hook** — a session `claim`s a scope (`repo:`,
  `path:<glob>`, `branch:<name>`); a conflicting Write/Edit, or a
  history-moving git command caught by a best-effort classifier, is **blocked
  with a message naming the holder.** Fail-open: a dead coordinator never stops
  work.
- **Consented, redacted awareness + `brief_me`** — with per-project consent
  (local-only, never committed), the daemon tails transcripts through
  agent-blackbox's redaction and distills a briefing bundle, so a joining
  helper gets context without raw conversation.
- **`assist`** — one command mints a fresh session, creates a git worktree on a
  new branch, briefs the helper, and launches a scoped headless `claude -p`
  that integrates via PR. Helpers are colleagues, not co-editors.

## What the dogfood proved — and what it deflated

conductor was tested on real work: two RFCs (`hindsight`, `agent-attest`)
authored as two coordinated sessions. The full record is
[conductor's `docs/CASE-STUDY.md`](https://github.com/hhagenbuch/conductor/blob/main/docs/CASE-STUDY.md).
The honest results, because the honest results are the point:

- **It worked where it mattered:** the shared platform-diagram update — the
  exact collision class from the incidents — was prevented by one lease and one
  bus message before either session touched the file.
- **The timing was real but modest:** the second RFC (a real `claude -p`
  helper) ran in the background while the first was written, so both finished
  in the time of the slower one (~297 s) instead of back-to-back (~384 s) —
  ~23%, with the unit of work doubled. Not "half the time"; "two jobs in the
  time of the longer one." Scoped to one run, not claimed as a controlled week.
- **It deflated one claim honestly:** for *cross-repo* work the leases are
  mostly informational — the enforcement teeth only bite when two sessions
  share a repo. And `assist`, built around a worktree of the parent's repo, is
  an *intra-repo* tool; the cross-repo worker ran over the bus instead. Both are
  now written down as design boundaries, not hidden.

That last bullet is the chapter's real lesson: **a coordinator earns trust by
reporting where it does not help.** conductor's case study says "here is the 23%
and here is the overhead and here is where the leases were only informational,"
which is the same doctrine every other repo in this playbook holds —
[honesty over silent degradation](../README.md) — applied to the tool that sits
above them all.
