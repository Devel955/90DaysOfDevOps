# Day 49 – DevSecOps: Add Security to Your CI/CD Pipeline

## What is DevSecOps?

DevSecOps means integrating security checks directly into the CI/CD pipeline so that vulnerabilities are caught automatically before code reaches production. Instead of a separate security team reviewing things weeks later, the pipeline itself blocks unsafe code, vulnerable images, and leaked secrets — every single time, without anyone having to remember to check. Security becomes part of the development process, not an afterthought.

---

## Task 1: Trivy Docker Image Scan

Added Trivy vulnerability scanner to `main-pipeline.yml` after Docker build.

### Step added to `main-pipeline.yml`
```yaml
- name: Scan Docker Image for Vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'anan623/github-actions-capstone:latest'
    format: 'table'
    exit-code: '0'
    severity: 'CRITICAL,HIGH'
```

### Base Image
`python:3.11-slim` (Debian 13.4)

### Scan Results
- Debian base image: 0 CVEs
- All Python packages: 0 CVEs
- Result: PASSED — clean image!

---

## Task 2: GitHub Secret Scanning

### How to Enable
1. Go to repo → Settings → Advanced Security
2. Enable Secret Protection
3. Enable Push Protection

### Result
Both were already enabled on the repo (public repos get this free).

### Secret Scanning vs Push Protection

| Feature | Secret Scanning | Push Protection |
|---------|----------------|-----------------|
| When it runs | After the push | Before the push |
| What it does | Detects secrets in repo | Blocks the push entirely |
| Who gets notified | Repo owner + service provider | Developer gets blocked |

### What happens if GitHub detects a leaked AWS key?
- GitHub sends an alert to repo owner via email
- GitHub notifies AWS directly — AWS can auto-revoke the key
- Alert appears in Security tab under Secret scanning alerts
- With Push Protection enabled, the push would have been blocked before reaching the repo

---

## Task 3: Dependency Review on PRs

Added dependency review to `pr-pipeline.yml`.

### Step added
```yaml
- name: Check Dependencies for Vulnerabilities
  uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: critical
```

### Result
PR ran successfully — No vulnerabilities or license issues found!

### How it works
- Runs only on pull_request events
- Checks new packages against GitHub Advisory Database
- Fails PR if any new dependency has a CRITICAL CVE
- Prevents vulnerable packages from reaching main branch

---

## Task 4: Workflow Permissions

Added least-privilege permissions to 2 workflow files.

### main-pipeline.yml
```yaml
permissions:
  contents: read
```

### pr-pipeline.yml
```yaml
permissions:
  contents: read
  pull-requests: write
```

### Why limit permissions?
If a third-party action is compromised, broad write access could allow it to:
- Delete or modify source code
- Push malicious commits
- Steal secrets via environment variables
- Modify workflow files for persistent access

Limiting to `contents: read` means even a compromised action can only read code.

---

## Task 5: Full Secure Pipeline
```
PR opened
    ↓
Build & Test ✅
    ↓
Dependency Vulnerability Check ✅  (NEW - Day 49)
    ↓
PR checks pass or fail

Push to main
    ↓
Build & Test ✅
    ↓
Docker Build & Push ✅
    ↓
Trivy Image Scan ✅  (NEW - Day 49)
(0 CVEs found on python:3.11-slim)
    ↓
Deploy to Production ✅
(manual approval required)

Always Active
    ↓
GitHub Secret Scanning ✅  (NEW - Day 49)
    ↓
Push Protection ✅  (NEW - Day 49)
```

---

## Key Learnings

| Concept | What I Learned |
|---------|---------------|
| Trivy | Scans Docker images for CVEs in one uses line |
| Secret Scanning | GitHub automatically detects leaked API keys |
| Push Protection | Blocks commits with secrets before they reach repo |
| Dependency Review | Catches vulnerable packages in PRs before merge |
| Permissions | Always use contents read unless write is needed |
| Shift Left | Catching issues in PRs is faster than fixing in production |

---

## What I Would Add Next

- SARIF upload — send Trivy results to GitHub Security tab
- OIDC authentication — replace Docker Hub tokens with short-lived tokens
- CodeQL scanning — static analysis for Python code
- Pinned SHAs — pin all Actions to commit SHAs
- Dependabot — auto-update vulnerable dependencies
- exit-code 1 — hard-fail pipeline on CRITICAL CVEs
