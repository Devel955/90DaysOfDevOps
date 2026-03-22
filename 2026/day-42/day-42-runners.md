# Day 42 – GitHub Actions Runners: GitHub-Hosted & Self-Hosted

## What is a Runner?
A runner is a machine that executes jobs in a GitHub Actions workflow.
- **GitHub-Hosted** → GitHub manages the machine
- **Self-Hosted** → You manage your own machine/server

---

## Task 1: GitHub-Hosted Runners (Multi-OS)

### Workflow: `multi-os.yml`
Three jobs running **in parallel** on different operating systems:

```yaml
jobs:
  ubuntu-job:
    runs-on: ubuntu-latest
  windows-job:
    runs-on: windows-latest
  macos-job:
    runs-on: macos-latest
```

### Result:
✅ ubuntu-job — Success (5s)  
✅ windows-job — Success (8s)  
✅ macos-job — Success (3s)  

> **Note:** A GitHub-hosted runner is a virtual machine provisioned by GitHub on demand.
> GitHub manages everything — spin up, run, destroy. You do nothing.

---

## Task 2: Pre-installed Tools on `ubuntu-latest`

```bash
Docker:  Docker version 26.x.x
Python:  Python 3.12.x
Node:    v20.x.x
Git:     git version 2.x.x
```

> **Why it matters:** No need to install common tools manually.
> Saves time, bandwidth, and keeps workflows simple.

Full list: https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md

---

## Task 3: Self-Hosted Runner Setup

### Steps followed:
1. GitHub → Settings → Actions → Runners → New self-hosted runner
2. Selected **Linux** as OS
3. Ran setup on **AWS EC2 instance**

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.322.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-linux-x64-2.322.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.322.0.tar.gz
./config.sh --url https://github.com/Devel955/github-actions-practice --token YOUR_TOKEN
./run.sh
```

### ✅ Runner Status: Idle (Green Dot)
```
Runner:  ip-172-31-45-248
Config:  Linux x64
Labels:  self-hosted | Linux | X64 | my-linux-runner
Status:  🟢 Idle
```

> **Screenshot:** Add your Runners settings page screenshot here

---

## Task 4: Workflow on Self-Hosted Runner

### File: `.github/workflows/self-hosted.yml`
```yaml
jobs:
  self-hosted-job:
    runs-on: [self-hosted, linux, my-linux-runner]
    steps:
      - name: Hostname print karo
        run: echo "Hostname: $(hostname)"

      - name: Working directory
        run: echo "PWD: $(pwd)"

      - name: File banao
        run: |
          FILE="$HOME/runner-demo-$(date +%Y%m%d-%H%M%S).txt"
          echo "Created by GitHub Actions self-hosted runner" > "$FILE"
          echo "Hostname: $(hostname)" >> "$FILE"
          echo "Run ID: $GITHUB_RUN_ID" >> "$FILE"
          cat "$FILE"
```

### ✅ Verification — File on my machine:
```bash
$ cat ~/runner-demo-20260322-174331.txt

Created by GitHub Actions self-hosted runner
Hostname: ip-172-31-45-248
Run ID: 23408757552
Date: Sun Mar 22 17:43:31 UTC 2026
```

> **Screenshot:** Add your job run screenshot here

---

## Task 5: Runner Labels

Label added: `my-linux-runner`

```yaml
runs-on: [self-hosted, linux, my-linux-runner]
```

> **Why labels are useful:**
> When you have multiple self-hosted runners, labels help route jobs
> to the right machine. Example:
> - `[self-hosted, gpu]` → GPU machine ke liye
> - `[self-hosted, large-mem]` → High memory machine ke liye
> - `[self-hosted, us-east]` → Region specific machine ke liye

---

## Task 6: GitHub-Hosted vs Self-Hosted

| Feature | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Who manages it?** | GitHub | You |
| **Cost** | Free (limits apply) | Your own hardware/VM cost |
| **Pre-installed tools** | Rich toolset ready | Only what you install |
| **Good for** | Standard CI/CD, open source | Private network, GPU, custom hardware |
| **Security concern** | GitHub controls environment | Untrusted code runs on your machine |
| **Startup time** | 10–30 seconds | Instant (always on) |
| **Persistence** | Destroyed after job | Files persist between runs |

---

## Permanent Service Setup

Runner as a background service (survives terminal close + machine restart):

```bash
cd actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

Output:
```
Active: active (running) ✅
Connected to GitHub ✅
```

---

## Key Learnings

1. GitHub-hosted runners → Zero setup, GitHub handles everything
2. Self-hosted runners → Full control, your own hardware
3. Labels → Route jobs to specific machines
4. Permanent service → Runner survives restarts
5. Security → Never use self-hosted on public repos with fork PRs

---

*Day 42/90 – #90DaysOfDevOps #GitHubActions #DevOps #AWS*
