# Runbook 1 — Verify, ship, and finish the two foundational repos

> **Produced:** an agent runtime (a reactive tool-calling service on Spring WebFlux) and an eval harness (golden datasets, deterministic assertions, LLM-as-judge, exit-code CI gate). **Repos:** https://github.com/hhagenbuch/spring-ai-agent-starter and https://github.com/hhagenbuch/agent-evals.
> **What the review cycle caught:** a streamed final turn that dropped conversation memory, an error path that bricked a session, and an MCP `tools/call` with no timeout that could poison the client — all fixed or documented as known limitations.

**For:** an AI agent with shell access.
**Goal:** take the two scaffolded repos from "written but never compiled" to "public with green CI", then implement the roadmap items as separate commits over multiple sessions.

## Context

- `spring-ai-agent-starter` — reactive tool-calling agent service (Spring Boot 3.3, WebFlux, Java 21). Core: `AgentLoop` (bounded model→tools→model recursion), `ToolRegistry` (auto-discovers `AgentTool` beans), `AnthropicClient` (Messages API + retry/backoff behind the `LlmClient` interface), `ConversationMemory` (in-memory per-session).
- `agent-evals` — eval harness CLI (plain Java 21, Jackson only, shaded jar). Core: YAML datasets → `TargetSystem` (Http/Echo) → assertions (`contains`/`not_contains`/`regex` deterministic; `judge` = LLM scored 1–5, **skipped not failed** when `ANTHROPIC_API_KEY` is absent) → markdown report + exit-code gate.
- Both have git history already (7 logical commits each). Working trees are clean.
- **The code has NEVER been compiled** (the scaffolding sandbox had no JDK 21/Maven). Assume there are compile errors to fix.
- The repos cross-reference each other in their READMEs, and `agent-evals`' `HttpTarget` speaks the starter's `POST /api/chat` contract: request `{"sessionId","message"}` → response `{"sessionId","reply"}`. Preserve this contract.

## Guardrails (non-negotiable)

1. Only touch the two repos named above.
2. Push over **HTTPS** only. Verify with `gh auth status` that the authenticated account is the intended personal account.
3. A forbidden-string grep (say, no internal codenames) must return 0 before every push — enforce it as a hard gate: `grep -ri "<forbidden-codename>" --exclude-dir=.git .` must return nothing.
4. Never commit secrets. API keys come from `ANTHROPIC_API_KEY` env only.
5. When fixing compile errors, make the **minimal** fix; don't restructure. The architecture and README claims are the product — keep code and README consistent (if you change behavior, update the README in the same commit).
6. Preserve commit hygiene: imperative subjects, one concern per commit.

## Prerequisites check

```bash
java -version    # need 21+
mvn -version
gh auth status   # logged in as the intended account
echo ${ANTHROPIC_API_KEY:+set}   # optional; needed only for live smoke + judge evals
```

If `gh` is missing: `brew install gh && gh auth login` (HTTPS, browser auth).

## Phase 1 — Make both builds green

For each repo, in order:

```bash
mvn -q verify   # in spring-ai-agent-starter
mvn -q verify   # in agent-evals
```

Fix errors until `verify` passes. Likely failure points to check first:

- `@DefaultValue` import must be `org.springframework.boot.context.properties.bind.DefaultValue`.
- `AgentProperties` record binding: constructor binding on records with `@DefaultValue` requires `@ConfigurationPropertiesScan` (present on `AgentStarterApplication`) — check property names in `application.yml` kebab-case match record components (`api-key` → `apiKey`, etc.).
- `AgentStarterApplicationTest` (`@SpringBootTest`) must load with **no** `ANTHROPIC_API_KEY` set. If it fails on the WebClient bean or property binding, fix the config, not the test — the "tests need no key" claim is a README selling point.
- In `AnthropicClient`, `Retry.backoff(...)` needs `reactor.util.retry.Retry`; `e.getStatusCode().value()` is correct for Spring 6 (`HttpStatusCode`).
- agent-evals: shade plugin runs at `package`; if `verify` complains about the main class, the path is `io.github.hhagenbuch.evals.EvalRunner`.
- agent-evals `DatasetLoaderTest` reads `datasets/smoke.yaml` relative to the repo root — surefire's working dir is the module root, so this should pass as-is; if not, fix via `Path.of("datasets/smoke.yaml")` resolution, not by moving the file.

Then run the harness self-test end-to-end (no key needed):

```bash
java -jar target/agent-evals-0.1.0-SNAPSHOT.jar datasets/smoke.yaml
# expect: 2/2 cases passed, exit 0, eval-report.md written
```

Commit fixes per repo: `git commit -m "Fix compile/test issues found on first real build"` (or more specific if the fix is interesting).

## Phase 2 — Live smoke test (needs ANTHROPIC_API_KEY)

Skip this phase if no key is available, and note that in the final report.

```bash
export ANTHROPIC_API_KEY=...   # from the environment — never echo or log it
mvn spring-boot:run &          # wait for startup on :8080
curl -s localhost:8080/api/chat -H 'content-type: application/json' \
  -d '{"message": "What is 973 * 481? Use your calculator."}'
```

Expected: reply contains `468013` (the model must have used the tool — the calculator only handles binary expressions, so `973 * 481` works). If the API errors: check the model id in `application.yml` is valid for the account; a 404 model error means pick the current Sonnet id from https://docs.claude.com/en/docs/about-claude/models.

Then run the real eval loop against it:

```bash
java -jar target/agent-evals-0.1.0-SNAPSHOT.jar datasets/customer-support.yaml
```

Expected: 3/3 with judge assertions scored. If `grounded-math` fails on the `contains 468013` assertion, debug the agent (tool schema, loop), not the dataset. Kill the server afterward.

## Phase 3 — Publish

```bash
gh repo create hhagenbuch/spring-ai-agent-starter --public --source=. --push \
  --description "Production-shaped agentic service in Java: bounded reactive tool-calling loop on Spring WebFlux"
gh repo edit hhagenbuch/spring-ai-agent-starter \
  --add-topic ai-agents --add-topic llm --add-topic spring-boot --add-topic webflux --add-topic java --add-topic anthropic

gh repo create hhagenbuch/agent-evals --public --source=. --push \
  --description "JUnit for LLM agents: golden datasets, deterministic assertions, LLM-as-judge, CI gates"
gh repo edit hhagenbuch/agent-evals \
  --add-topic llm-evaluation --add-topic ai-agents --add-topic testing --add-topic java --add-topic ci
```

Verify CI: `gh run watch` in each repo (or `gh run list`). Both workflows must go green — `CI` (starter) and `Eval gate` (evals, which also executes the smoke dataset against the shaded jar). Fix-and-push until green. Green CI is the acceptance bar for Phases 1–3.

## Phase 4 — Roadmap implementation (separate commits over subsequent sessions)

Do these as separate, well-messaged commits/PRs — do not batch them.

### 4a. starter: SSE streaming (`GET /api/chat/stream`)

`AnthropicClient` gains `chatStream(...)` using `"stream": true` and `bodyToFlux(String)` over the SSE lines (or `ServerSentEvent` decoding); parse `content_block_delta` events into a `Flux<String>` of text deltas. Simplest correct scope: stream **only the final answer turn** — run the tool loop to completion with the existing non-streaming path, then stream the last model call. **Watch the memory seam:** the streamed final turn must still be written back into `ConversationMemory` or the next turn loses context (this was a real review finding). Controller returns `Flux<ServerSentEvent<String>>` with `MediaType.TEXT_EVENT_STREAM`. Add a test with a fake that emits a 3-chunk Flux. Update README roadmap checkbox + add a curl example (`curl -N`).

### 4b. evals: `--min-pass-rate` flag

`EvalRunner`: parse `--min-pass-rate 0.9` (default `1.0`); exit 0 iff `passed/total >= threshold`. Update usage string, README, and add a runner test for the threshold boundary.

### 4c. starter: MCP client

Add an `McpToolAdapter`: connect to an MCP server over stdio (JSON-RPC 2.0: `initialize`, `tools/list`, `tools/call`), wrap each discovered MCP tool as an `AgentTool` so the registry treats it identically to local tools. Config: `agent.mcp-servers[0].command: ...` list in `application.yml`. **Put a timeout on `tools/call`** — a hung or misbehaving server must not block the agent loop indefinitely and poison the client (this was a review finding). Test against a trivial stdio MCP echo server included under `src/test`. This makes the README claim "mount any MCP server's tools" true.

### 4d. evals: trajectory assertions

New assertion `type: tool_called` / `value: calculator`. Requires the target to expose the tool trace: add `toolsUsed: [..]` to the starter's `ChatResponse` (AgentLoop already knows the calls it executed — collect names per run), have `HttpTarget` pass it through, assert on it. Cross-repo commit pair; update both READMEs. This answers "how do you know the agent did the right thing, not just said the right thing."

### 4e. evals: parallel cases + judge ensembling (lowest priority)

Parallelize case execution with a virtual-thread executor (`Executors.newVirtualThreadPerTaskExecutor()`). Judge ensembling: run the judge k=3 times, take the median score.

## Handling the error path

The chat error path must fail a single turn without corrupting session state — an exception mid-turn should surface an error to the caller and leave `ConversationMemory` consistent, not leave the session unusable for every subsequent request (this was a real review finding; the fix was to isolate per-turn failure and document the recovery semantics).

## Acceptance checklist

- [ ] `mvn verify` green in both repos locally
- [ ] Both repos public, CI green, descriptions + topics set
- [ ] Forbidden-string grep clean on every push
- [ ] Smoke eval (`smoke.yaml`) passes via the shaded jar
- [ ] Live smoke (Phase 2) done or explicitly noted as skipped
- [ ] Roadmap items landed as individual commits, READMEs kept in sync (checkboxes flipped as delivered)
- [ ] Streamed turn preserves memory; MCP `tools/call` has a timeout; error path does not brick a session
- [ ] Report back: what was fixed in Phase 1, CI links, which roadmap items are done/remaining
