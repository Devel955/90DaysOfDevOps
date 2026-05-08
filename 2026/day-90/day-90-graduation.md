# 🎓 Day 90 — Graduation: My 90 Days of DevOps Journey

> **Challenge:** #90DaysOfDevOps by [@TrainWithShubham](https://github.com/TrainWithShubham)  
> **Duration:** 90 Days  
> **Status:** ✅ Complete

---

## 📅 The 90-Day Timeline

| Week | Days | Topic | Key Tools |
|------|------|--------|-----------|
| 1–2 | 1–13 | Linux Fundamentals | `bash`, `chmod`, `lvm`, `systemd` |
| 3 | 14–15 | Networking | DNS, IP, subnets, ports |
| 3–4 | 16–21 | Shell Scripting | Bash, functions, cron |
| 4–5 | 22–28 | Git & GitHub | branching, GitHub CLI, advanced git |
| 5–6 | 29–37 | Docker | Dockerfile, Compose, multi-stage builds |
| 6–7 | 38–49 | CI/CD & GitHub Actions | YAML, runners, secrets, DevSecOps |
| 7–8 | 50–58 | Kubernetes | Pods, Deployments, RBAC, Namespaces |
| 8–9 | 59–67 | Terraform (TerraWeek) | providers, state, modules, workspaces |
| 9–10 | 68–72 | Ansible | inventory, playbooks, roles, Vault |
| 10–11 | 73–77 | Observability | Prometheus, Grafana, Loki, OpenTelemetry |
| 11 | 78–80 | Helm | charts, values, subcharts, CI/CD hooks |
| 12 | 81–83 | Amazon EKS | Terraform for EKS, Gateway API, IRSA, HPA |
| 12–13 | 84–86 | ArgoCD & GitOps | sync waves, App of Apps, full pipeline |
| 13 | 87–89 | Agentic AI for DevOps | LLM agents, ReAct, MCP, KubeHealer, Temporal |
| 13 | **90** | **Grand Finale** | **Everything connected** |

---

## 💡 My Top 5 "Aha!" Moments

1. **GitOps clicked on Day 84** — Realizing that Git is not just version control but the *single source of truth* for your entire infrastructure. ArgoCD constantly reconciling the cluster state to match the repo was mind-blowing.

2. **Docker layers on Day 31** — Understanding that every `RUN` instruction creates a new layer and how multi-stage builds dramatically reduce image size changed how I think about packaging.

3. **Terraform state on Day 61** — The moment I understood *why* state files exist and what happens when they go out of sync. Never take `terraform.tfstate` for granted again.

4. **Kubernetes RBAC on Day 57** — Connecting Linux file permissions concepts (from Day 3) to Kubernetes RBAC. The mental model from Day 1 paid off on Day 57.

5. **AI agents on Day 88** — Watching a KubeHealer agent autonomously detect a `CrashLoopBackOff`, query logs, diagnose the root cause, and propose a fix — without any human input. The future is here.

---

## 😤 The Hardest Day

**Day 65 — Terraform Workspaces + Remote State**

Managing multiple environments with workspaces while keeping remote state isolated in S3 with DynamoDB locking was genuinely painful. The error messages were cryptic, the state got corrupted once, and I had to manually inspect the state file JSON to figure out what went wrong.

How I pushed through: re-read the Terraform docs, blew away the broken state in a test environment, and rebuilt from scratch while documenting every step. That rebuild became my best notes from the entire challenge.

---

## 📊 Skills Inventory

| Skill | Days | Confidence (1–5) |
|-------|------|-----------------|
| Linux command line | 1–13 | ⭐⭐⭐⭐⭐ |
| Shell scripting | 16–21 | ⭐⭐⭐⭐ |
| Git & GitHub | 22–28 | ⭐⭐⭐⭐⭐ |
| Docker | 29–37 | ⭐⭐⭐⭐⭐ |
| CI/CD (GitHub Actions) | 38–49 | ⭐⭐⭐⭐ |
| Kubernetes | 50–58 | ⭐⭐⭐⭐ |
| Terraform | 59–67 | ⭐⭐⭐⭐ |
| Ansible | 68–72 | ⭐⭐⭐ |
| Observability (Prometheus, Grafana, Loki) | 73–77 | ⭐⭐⭐⭐ |
| Helm | 78–80 | ⭐⭐⭐⭐ |
| Amazon EKS | 81–83 | ⭐⭐⭐ |
| ArgoCD / GitOps | 84–86 | ⭐⭐⭐⭐ |
| Agentic AI for DevOps | 87–89 | ⭐⭐⭐ |

---

## 🏗️ The End-to-End Pipeline I Built

```
Developer writes code on Linux (Days 1–13)
    ↓ git push (Days 22–28)
GitHub Actions triggers (Days 38–49)
    ↓ builds Docker image (Days 29–37)
    ↓ pushes to DockerHub
    ↓ updates K8s manifest in Git
ArgoCD detects change (Days 84–86)
    ↓ syncs to EKS cluster (Days 81–83)
EKS cluster provisioned by Terraform (Days 59–67)
    ↓ configured by Ansible (Days 68–72)
    ↓ app deployed via Helm chart (Days 78–80)
Prometheus + Grafana + Loki monitors it all (Days 73–77)
    ↓ if something breaks
AI agent (KubeHealer) diagnoses and proposes fix (Days 87–89)
    ↓ fix committed to Git
    ↓ ArgoCD syncs → cycle continues
```

---

## 🚀 What I'm Learning Next

- [ ] **CKA (Certified Kubernetes Administrator)** — aiming for it in 60 days
- [ ] **Service Mesh** — Istio for zero-trust networking between microservices
- [ ] **HashiCorp Vault** — proper secrets management, not just K8s secrets
- [ ] **Terragrunt** — DRY Terraform for multi-environment setups
- [ ] **Chaos Engineering** — Litmus Chaos on EKS to test resilience
- [ ] **FinOps** — cloud cost visibility with Kubecost

---

## 💬 Advice for Someone Starting Day 1 Tomorrow

1. **Don't skip Linux.** Every single tool in this challenge runs on Linux. Days 1–13 feel slow but they are the foundation everything else builds on.

2. **Break things intentionally.** The best learning happened when something broke and I had to fix it. Don't just follow happy-path tutorials.

3. **Write notes in your own words.** Pasting from docs doesn't stick. Explaining a concept like you'd teach it to someone else is what creates real understanding.

4. **Do the projects, not just the reading.** The AI-BankApp project in the last 13 days tied everything together more than any single topic day.

5. **Show up every day.** Not every day will be exciting. Day 71 (Ansible templates) was genuinely boring. Show up anyway. Consistency beats motivation.

6. **Ask questions publicly.** Post your blockers on LinkedIn or Twitter/X. The community responds fast and you'll learn from the answers even when someone else asks.

---

## 🙏 Thank You

Thank you to **Shubham Londhe** and the **TrainWithShubham** community for building and maintaining this challenge. The structured curriculum, real-world project (AI-BankApp), and community support made 90 days feel achievable.

---

*Generated on Day 90 of the #90DaysOfDevOps challenge*  
*#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham*
