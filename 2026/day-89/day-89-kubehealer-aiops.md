# Day 89 — Production AI Agents: KubeHealer and AIOps

## Overview

Today's challenge moved from agents that *diagnose* to an agent that *fixes* — autonomously, durably, and safely. KubeHealer scans a Kubernetes cluster for broken pods, uses Claude to reason about root causes, proposes targeted fixes, and applies them only after human approval. Temporal makes it crash-resistant: if the agent dies mid-repair, it resumes exactly where it left off.

This is AIOps — not a chatbot, but a system that watches, reasons, and acts.

---

## Task 1: AIOps Principles and Production Guardrails

### What Is AIOps?

AIOps means using AI to automate IT operations: monitoring, diagnosis, and remediation. The goal is not to replace humans but to augment them — letting the agent handle routine, well-understood failures (image typos, resource limit misconfigurations) while escalating anything that requires human judgment.

### The 6 Production Guardrails

Every production AI agent that touches infrastructure must implement these:

| Guardrail | Why It Matters | KubeHealer Implementation |
|---|---|---|
| **Human approval** | Agents must not make destructive changes without permission | Workflow pauses, presents all proposed fixes, waits for `yes/no` |
| **Scope limits** | Agents should only operate in allowed namespaces | Restricted to `default` namespace; cannot touch `kube-system` |
| **Audit trail** | Every action must be recorded for accountability | Temporal records every activity, every Claude call, every kubectl command |
| **Rollback capability** | Every fix must be reversible | Uses `kubectl patch` (targeted change), not delete-and-recreate |
| **Timeout and retry limits** | Agents must not loop forever | Max retries per pod, timeout after 5 minutes |
| **Escalation path** | When the agent cannot fix it, a human must be notified | `config-app` diagnosed but escalated — agent reports it cannot proceed |

### When to Use AI Agents vs Traditional Automation

| Use AI Agents When | Use Traditional Automation When |
|---|---|
| Problem requires reasoning (diagnose unknown errors) | Problem has a known, fixed solution |
| Multiple possible causes and fixes exist | One cause, one fix (if X then Y) |
| Natural language output helps a human understand | No human in the loop |
| Root cause is ambiguous from raw logs | Scaling, restarts, scheduled deploys |

---

## Task 2: KubeHealer Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    KubeHealer System                    │
│                                                         │
│  ┌──────────┐    ┌───────────────┐    ┌─────────────┐  │
│  │ starter  │───►│    Temporal   │───►│   worker    │  │
│  │  .py     │    │  (durable     │    │   .py       │  │
│  └──────────┘    │  execution)   │    └──────┬──────┘  │
│                  └───────────────┘           │         │
│                         │                   │         │
│                   Audit trail          ┌─────▼──────┐  │
│                   Crash recovery       │  Claude    │  │
│                   Workflow replay      │  Sonnet 4  │  │
│                                        └─────┬──────┘  │
│                                              │         │
│                                        ┌─────▼──────┐  │
│                                        │  kubectl   │  │
│                                        │  (patch)   │  │
│                                        └────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Components:**

- **starter.py** — triggers the Temporal workflow
- **Temporal** — durable execution engine; records every step, enables crash recovery
- **worker.py** — runs the agent activities (scan, diagnose, fix)
- **Claude Sonnet 4** — reasons about root causes and proposes fixes (stronger model needed for safe infrastructure changes)
- **kubectl patch** — applies targeted, reversible fixes

Why Claude Sonnet 4 and not a local model? Production infrastructure changes require stronger reasoning to avoid proposing incorrect fixes. The cost of a wrong `kubectl patch` is much higher than an API call.

---

## Task 3: The Three Broken Apps

### App 1 — Image Typo (`web-app`) ✅ Fixable

```yaml
image: ngnix:latest   # typo — should be nginx:latest
```

**Status:** `ImagePullBackOff`
**Agent diagnosis:** "Image `ngnix:latest` does not exist in the registry. This is a typo — the correct image is `nginx:latest`."
**Proposed fix:** `kubectl patch pod web-app` with corrected image name
**Outcome:** Fixed automatically after approval

### App 2 — OOM Crash (`memory-app`) ✅ Fixable

```yaml
resources:
  limits:
    memory: "1Mi"   # far too low for nginx
```

**Status:** `CrashLoopBackOff` (OOMKilled)
**Agent diagnosis:** "Container is being killed by the OOM killer. Memory limit of 1Mi is insufficient for the workload."
**Proposed fix:** Increase memory limit to `128Mi`
**Outcome:** Fixed automatically after approval

### App 3 — Missing ConfigMap (`config-app`) ❌ Escalated

```yaml
envFrom:
- configMapRef:
    name: app-config   # ConfigMap does not exist
```

**Status:** `CreateContainerConfigError`
**Agent diagnosis:** "Pod references ConfigMap `app-config` which does not exist in the `default` namespace."
**Proposed fix:** None — creating arbitrary ConfigMaps requires human decision about the correct values
**Outcome:** Reported as unfixable, escalated to human

This is a deliberate design choice. A good agent knows its limits.

---

## Task 4: Agent Approval Flow

After scanning and diagnosing all three pods, KubeHealer presents:

```
Found 3 broken pods.

Proposed fixes:
1. web-app: Fix image typo (ngnix:latest → nginx:latest)
2. memory-app: Increase memory limit (1Mi → 128Mi)
3. config-app: CANNOT FIX — needs manual ConfigMap creation (app-config)

Approve all fixes? [yes/no]:
```

The workflow **pauses** here — this is a Temporal signal. The process does not spin or poll; it simply waits until a human responds. In production, this pause point could be a Slack message, a PagerDuty alert, or a web UI approval button.

After typing `yes`:
- `web-app` patched with correct image → `Running`
- `memory-app` patched with increased memory → `Running`
- `config-app` skipped, escalation logged in Temporal history

```bash
kubectl get pods
NAME         READY   STATUS    RESTARTS
web-app      1/1     Running   0
memory-app   1/1     Running   0
config-app   0/1     CreateContainerConfigError   0   # expected
```

---

## Task 5: Crash Recovery with Temporal

### The Test

1. Deploy all three broken apps
2. Start the worker: `python3 worker.py`
3. Trigger the workflow: `python3 starter.py`
4. While the agent is mid-diagnosis — **kill the worker**
5. Restart the worker: `python3 worker.py`

### What Happens Without Temporal

All in-memory state is lost. The agent restarts from scratch, re-scanning and re-diagnosing everything. For long-running infrastructure repairs, this means lost work and potential inconsistency (some fixes applied, some not).

### What Happens With Temporal

Temporal replays the completed activities from its event history. Steps already executed (scan, diagnose) are skipped — their recorded outputs are replayed directly. The workflow resumes at the exact point of interruption — the approval prompt — as if the crash never happened.

### Temporal UI Workflow View

At `http://localhost:8233`, the workflow history shows:

```
WorkflowExecutionStarted
ActivityTaskScheduled  → scan_cluster
ActivityTaskCompleted  → [web-app, memory-app, config-app]
ActivityTaskScheduled  → diagnose_pod (web-app)
ActivityTaskCompleted  → "Image typo detected..."
ActivityTaskScheduled  → diagnose_pod (memory-app)
ActivityTaskCompleted  → "OOMKilled detected..."
ActivityTaskScheduled  → diagnose_pod (config-app)
ActivityTaskCompleted  → "Missing ConfigMap..."
[WORKER CRASH — replayed from here on restart]
WorkflowExecutionSignaled → human_approval: yes
ActivityTaskScheduled  → apply_fix (web-app)
ActivityTaskCompleted  → patched successfully
ActivityTaskScheduled  → apply_fix (memory-app)
ActivityTaskCompleted  → patched successfully
WorkflowExecutionCompleted
```

Every Claude call, every kubectl command, every decision — all recorded automatically. This is the audit trail.

---

## Task 6: The 3-Day Agentic AI Journey

| Day | What Was Built | Pattern |
|---|---|---|
| 87 | Docker Error Explainer + Docker Agent | Basic LLM → ReAct Agent |
| 88 | Multi-tool Agent + MCP Server + CI/CD Analyzer | Multi-domain tools, MCP protocol |
| 89 | KubeHealer — production self-healing agent | Temporal durability, human approval, guardrails |

### The Evolution

```
Day 87: LLM explains errors (passive observation)
         ↓
Day 88: Agent diagnoses across Docker/K8s/CI (autonomous investigation)
         ↓
Day 89: Agent diagnoses AND fixes with approval (autonomous action)
```

### How Agentic AI Connects to the Full 90-Day Challenge

| Days | Topic | Connection |
|---|---|---|
| 29–37 | Docker | Docker tools in Module 2 wrap the same `docker` commands learned here |
| 40–49 | GitHub Actions | CI/CD Analyzer diagnoses the pipelines built during this section |
| 50–67 | Kubernetes | K8s tools and KubeHealer use `kubectl` on the clusters set up here |
| 73–77 | Observability | Agents could query Prometheus/Loki for metric-based diagnosis |
| 84–86 | ArgoCD | An agent could trigger ArgoCD syncs or rollbacks as a fix action |

---

## Key Principles for Production AI Agents

1. **Tools are just CLI wrappers** — any command you run can become an agent tool
2. **The ReAct pattern is universal** — works for any domain: Docker, K8s, CI/CD, Terraform, AWS
3. **MCP standardises tool access** — write once, use from any compatible client
4. **Guardrails are not optional** — approval, scope limits, and audit trails are non-negotiable in production
5. **Durability matters** — Temporal prevents lost state during infrastructure changes
6. **Know when NOT to use AI** — simple if/then automation is better for well-understood, deterministic problems
7. **Good agents know their limits** — `config-app` was escalated, not guessed at

---

## Cleanup

```bash
kind delete cluster --name kubehealer-demo
# Stop Temporal (Ctrl+C)
deactivate
```

---

*Day 89 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
