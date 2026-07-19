# Runbook 4 — `agent-operator`: treat agents as Kubernetes workloads

> **Produced:** `agent-operator`, a Kubernetes operator that rolls out prompt/model-version changes as eval-gated canaries with automatic rollback on regression. **Repo:** https://github.com/hhagenbuch/agent-operator.
> **What the review cycle caught:** the eval-gate image was hardcoded (so the gate couldn't be pointed at a different eval build) and a stuck gate had no timeout (a hung eval Job would wedge a rollout forever) — both fixed (PR #4).

**For:** an AI agent with shell access, plus a human reviewing design decisions.
**Goal:** ship `agent-operator` — a Kubernetes operator where deploying a new prompt or model version works like deploying code: canary, eval gate, auto-rollback.
**Builds on:** Runbook 1. The operator *deploys* the agent runtime and *gates* with the eval harness — the three repos closing into a platform is the point.

## The pitch (top of README)

> Prompts and model versions change agent behavior as much as code changes
> service behavior — but most teams deploy them with a config edit and a
> prayer. `agent-operator` makes agent changes first-class Kubernetes
> deployments: declare an `Agent`, roll out a `PromptVersion` as a canary,
> gate promotion on an eval suite run in-cluster, and roll back automatically
> on regression. Prompts have SLOs now.

## Why this matters

- It's the synthesis: Kubernetes platform depth × agent engineering × eval discipline. Most teams are duct-taping this with bash.
- "GitOps for prompts" is a one-line concept that survives being repeated.
- A Java operator is itself a differentiator (the ecosystem defaults to Go) and stays consistent with the rest of the stack.

## Design

### CRDs (`agents.hhagenbuch.io/v1alpha1`)

**`Agent`** — the long-running workload:

```yaml
apiVersion: agents.hhagenbuch.io/v1alpha1
kind: Agent
metadata:
  name: support-agent
spec:
  image: ghcr.io/hhagenbuch/spring-ai-agent-starter:0.1.0
  replicas: 2
  model: claude-sonnet-5
  apiKeySecretRef: { name: anthropic-key, key: api-key }
  activePromptVersion: support-v3          # managed by the operator, not humans
  evalGate:
    datasetConfigMap: support-golden-cases  # agent-evals YAML
    evalImage: ghcr.io/hhagenbuch/agent-evals:0.1.0   # configurable, not hardcoded
    minPassRate: "0.9"
    timeoutSeconds: 600                     # a gate that never finishes must fail, not hang
```

**`PromptVersion`** — immutable, versioned prompt/config change:

```yaml
apiVersion: agents.hhagenbuch.io/v1alpha1
kind: PromptVersion
metadata:
  name: support-v4
spec:
  agentRef: support-agent
  systemPrompt: |
    You are a support agent...
  model: claude-sonnet-5        # optional override — model bumps flow through the same gate
  rollout:
    strategy: EvalGatedCanary   # or Immediate (dev only)
    canaryWeight: 20            # % of traffic during canary (stretch goal; MVP = eval-only gate)
```

Status is where the product lives — `PromptVersion.status.phase`:
`Pending → Canary → Evaluating → Promoted | RolledBack`, with conditions
carrying the eval report summary (`evalPassRate: 0.94`, link to report).
`kubectl get promptversions` telling you *why* a prompt was rolled back is
the demo money-shot.

### Reconcile loops

**Agent controller:** ensures Deployment + Service + ConfigMap (rendered from
`activePromptVersion`) exist and match spec. Prompt content lives in a
ConfigMap named `{agent}-{promptversion}`; the pod template hashes it so a
promotion is a normal rolling update — you inherit k8s rollout/rollback
semantics for free instead of reinventing them. That design decision goes in
the README.

**PromptVersion controller** (the interesting one):

1. New `PromptVersion` → spin up a **canary Deployment** (1 replica) of the
   agent image with the new prompt ConfigMap, not yet receiving Service traffic.
2. Launch an **eval Job**: the eval-harness image (from `evalGate.evalImage` —
   never a hardcoded reference), dataset mounted from the ConfigMap,
   `--target http://{canary-service}/api/chat`, `--min-pass-rate` from the gate
   spec. (This is why Runbook 1's `--min-pass-rate` flag must exist first.)
3. Job exit 0 → **Promote**: patch `Agent.spec.activePromptVersion`, let the
   Agent controller roll the main Deployment; delete canary; phase `Promoted`.
4. Job exit 1 → **RolledBack**: delete canary, main Deployment untouched,
   attach the eval report (Job pod logs → status condition + Event). Emit
   k8s Events for every transition — `kubectl describe` should tell the story.
5. **Gate timeout:** the eval Job must have a bounded deadline
   (`evalGate.timeoutSeconds`). A Job that never completes transitions the
   PromptVersion to `RolledBack` with a timeout reason — a stuck gate is a
   failed gate, never an indefinitely-wedged rollout. This was a review finding.
6. Judge assertions need the API key; pass the `apiKeySecretRef` secret into
   the eval Job. Without it, the deterministic tier still gates (document this).

### Tech choices

- **Java Operator SDK** (`io.javaoperatorsdk:operator-framework`, latest 4.x/5.x) on plain Java 21 + Maven; JOSDK handles informers/requeue.
- Fabric8 CRD generator (`crd-generator-apt`) to emit CRD YAML from the Java model classes — single source of truth.
- Local target: **kind**. Everything must run on a laptop with `kind create cluster`.
- No webhook/cert-manager complexity in MVP — validation in the reconciler. Note it as a non-goal.

## Build phases

### Phase 0 — Design doc (half day)

`docs/DESIGN.md` before code: the two CRDs with full YAML, state machine,
reconcile sequence diagram, decisions (ConfigMap-hash promotion, Job-based
eval gate with a configurable image and a hard timeout, Java-not-Go rationale),
non-goals (traffic-split canary, multi-cluster, webhooks, HPA). Commit alone:
"Design: eval-gated deployments for agents".

### Phase 1 — Agent controller

Scaffold with JOSDK quickstart. `Agent` CRD + controller reconciling
Deployment/Service/ConfigMap. Unit-test reconciler logic with JOSDK's
mock-server support; e2e: `kind` + apply an Agent + curl the chat endpoint
(needs the starter image published — `mvn spring-boot:build-image` +
`kind load docker-image`, script it in `hack/`).

### Phase 2 — PromptVersion controller + eval gate

The state machine above. Watch Jobs for completion; parse pass/fail from exit
code, pull the report tail from pod logs into status. e2e demo test: apply a
good PromptVersion → `Promoted`; apply one with a sabotaged prompt ("always
answer in French") against an English-asserting dataset → `RolledBack` with
the failing report in `kubectl describe`. That sabotage demo is the GIF.

### Phase 3 — Polish

`kubectl get` printer columns (PHASE, PASSRATE, AGE), Events everywhere,
Helm chart or `kustomize` install, README with GIF + architecture diagram +
`kind`-based 5-minute quickstart, roadmap (traffic-weighted canary via
Gateway API, drift detection: nightly eval re-runs against the *live* prompt,
`ModelVersion` CRD).

## Guardrails

No employer names (the repo never says where any of this runs); HTTPS to the
intended account only; the forbidden-string grep must return 0 before every
push; design doc first; phases as separate commits. Clean room — no reuse of
any internal manifests.

## Prereqs

```bash
brew install kind kubectl helm
docker --version   # Docker Desktop running
```

## Acceptance checklist

- [ ] DESIGN.md committed before code
- [ ] `kind` quickstart: Agent up and answering in <5 min from clone
- [ ] Good PromptVersion → `Promoted`; sabotaged → `RolledBack` with report in status — both covered by e2e and the README GIF
- [ ] Eval-gate image is configurable; a hung gate times out to `RolledBack`
- [ ] CRD YAML generated from Java sources, printer columns set
- [ ] Public at github.com/hhagenbuch/agent-operator, forbidden-string grep clean
