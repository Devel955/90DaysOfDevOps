# Day 48 – GitHub Actions Capstone: End-to-End CI/CD Pipeline

## Task Overview
Today I built a complete, production-style CI/CD pipeline using everything learned from Day 40–47: workflows, triggers, secrets, Docker builds, reusable workflows, and advanced events.

---

## Task 1: Project Repo Setup

- **Repo:** [github-actions-capstone](https://github.com/Devel955/github-actions-capstone)
- **App:** Python FastAPI with `/health` endpoint
- **Dockerfile:** Multi-stage build using `python:3.11-slim`
- **Test:** `pytest` using FastAPI `TestClient`

### App — `app/main.py`
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok", "service": "github-actions-capstone"}
```

### Test — `app/test_main.py`
```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"
```

### Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Task 2: Reusable Workflow — Build & Test

**File:** `.github/workflows/reusable-build-test.yml`  
**Trigger:** `workflow_call`  
**Inputs:** `python_version` (string), `run_tests` (boolean)  
**Output:** `test_result` → `passed` or `skipped`
```yaml
name: Reusable - Build and Test
on:
  workflow_call:
    inputs:
      python_version:
        type: string
        default: "3.11"
      run_tests:
        type: boolean
        default: true
    outputs:
      test_result:
        value: ${{ jobs.build-test.outputs.test_result }}

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:
      test_result: ${{ steps.set-result.outputs.result }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python_version }}
      - run: pip install -r requirements.txt
      - if: ${{ inputs.run_tests }}
        run: python -m pytest app/test_main.py -v
      - id: set-result
        run: echo "result=passed" >> $GITHUB_OUTPUT
```

---

## Task 3: Reusable Workflow — Docker Build & Push

**File:** `.github/workflows/reusable-docker.yml`  
**Trigger:** `workflow_call`  
**Inputs:** `image_name`, `tag`  
**Secrets:** `docker_username`, `docker_token`  
**Output:** `image_url`
```yaml
name: Reusable - Docker Build and Push
on:
  workflow_call:
    inputs:
      image_name:
        type: string
      tag:
        type: string
    secrets:
      docker_username:
        required: true
      docker_token:
        required: true
    outputs:
      image_url:
        value: ${{ jobs.docker-build-push.outputs.image_url }}

jobs:
  docker-build-push:
    runs-on: ubuntu-latest
    outputs:
      image_url: ${{ steps.set-image-url.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.docker_username }}
          password: ${{ secrets.docker_token }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ inputs.image_name }}:${{ inputs.tag }}
      - id: set-image-url
        run: echo "url=${{ inputs.image_name }}:${{ inputs.tag }}" >> $GITHUB_OUTPUT
```

---

## Task 4: PR Pipeline

**File:** `.github/workflows/pr-pipeline.yml`  
**Trigger:** `pull_request` to `main` (opened, synchronize)  
**Jobs:** build-test + pr-comment  
**Note:** No Docker push on PRs
```yaml
name: PR Pipeline
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize]

jobs:
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.11"
      run_tests: true

  pr-comment:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - name: Print PR summary
        run: |
          echo "PR checks passed for branch: ${{ github.head_ref }}"
          echo "Test result: ${{ needs.build-test.outputs.test_result }}"
      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `PR checks passed for branch: ${{ github.head_ref }}`
            })
```

---

## Task 5: Main Branch Pipeline

**File:** `.github/workflows/main-pipeline.yml`  
**Trigger:** `push` to `main`  
**Jobs:** build-test → docker → deploy (manual approval)
```yaml
name: Main Branch Pipeline
on:
  push:
    branches: [main]

jobs:
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.11"
      run_tests: true

  docker:
    runs-on: ubuntu-latest
    needs: build-test
    outputs:
      image_url: ${{ steps.set-image-url.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - id: short-sha
        run: echo "sha=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT
      - run: echo "${{ secrets.DOCKER_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
      - name: Build and push image
        env:
          DOCKER_BUILDKIT: 0
        run: |
          docker build -t ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest .
          docker tag ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest \
            ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:sha-${{ steps.short-sha.outputs.sha }}
          docker push ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest
          docker push ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:sha-${{ steps.short-sha.outputs.sha }}
      - id: set-image-url
        run: echo "url=${{ vars.DOCKER_USERNAME }}/github-actions-capstone:sha-${{ steps.short-sha.outputs.sha }}" >> $GITHUB_OUTPUT

  deploy:
    runs-on: ubuntu-latest
    needs: docker
    environment: production
    steps:
      - run: |
          echo "Deploying image: ${{ needs.docker.outputs.image_url }} to production"
          echo "Triggered by: ${{ github.actor }}"
      - run: |
          echo "## Deployment Summary" >> $GITHUB_STEP_SUMMARY
          echo "- Image: ${{ needs.docker.outputs.image_url }}" >> $GITHUB_STEP_SUMMARY
          echo "- Actor: ${{ github.actor }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date -u)" >> $GITHUB_STEP_SUMMARY
```

---

## Task 6: Scheduled Health Check

**File:** `.github/workflows/health-check.yml`  
**Trigger:** cron `0 */12 * * *` + `workflow_dispatch`
```yaml
name: Scheduled Health Check
on:
  schedule:
    - cron: "0 */12 * * *"
  workflow_dispatch:

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.DOCKER_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
      - run: docker pull ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest
      - run: docker run -d --name app -p 8000:8000 ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest
      - run: sleep 5
      - id: health
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health)
          if [ "$STATUS" = "200" ]; then
            echo "result=PASSED" >> $GITHUB_OUTPUT
          else
            echo "result=FAILED" >> $GITHUB_OUTPUT
            exit 1
          fi
      - if: always()
        run: |
          echo "## Health Check Report" >> $GITHUB_STEP_SUMMARY
          echo "- Image: ${{ vars.DOCKER_USERNAME }}/github-actions-capstone:latest" >> $GITHUB_STEP_SUMMARY
          echo "- Status: ${{ steps.health.outputs.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date -u)" >> $GITHUB_STEP_SUMMARY
      - if: always()
        run: docker stop app && docker rm app
```

---

## Task 7: Pipeline Architecture & Badges

### Status Badges
![PR Pipeline](https://github.com/Devel955/github-actions-capstone/actions/workflows/pr-pipeline.yml/badge.svg)
![Main Pipeline](https://github.com/Devel955/github-actions-capstone/actions/workflows/main-pipeline.yml/badge.svg)
![Health Check](https://github.com/Devel955/github-actions-capstone/actions/workflows/health-check.yml/badge.svg)

### Pipeline Flow
```
PR opened
    ↓
Build & Test ✅
    ↓
PR Comment: "checks passed" ✅
(No Docker push on PRs)

Push to main
    ↓
Build & Test ✅
    ↓
Docker Build & Push ✅
(tags: latest + sha-xxxxxxx)
    ↓
Deploy to Production ✅
(manual approval required)

Every 12 hours
    ↓
Pull latest image
    ↓
Run container → curl /health
    ↓
PASSED/FAILED → Step Summary
```

### Docker Hub
Image: [anan623/github-actions-capstone](https://hub.docker.com/r/anan623/github-actions-capstone)  
Tags: `latest`, `sha-386f271`

---

## What I'd Add Next

| Improvement | Why |
|-------------|-----|
| Slack notifications | Alert on failure without checking GitHub |
| Multi-environment | staging → production promotion |
| Rollback workflow | Re-deploy previous SHA on demand |
| Trivy security scan | Fail on CRITICAL CVEs |
| Matrix testing | Python 3.10, 3.11, 3.12 parallel |
| Dependabot | Auto-update Actions versions |
| pip caching | Faster build times |

---

## Key Learnings

- Reusable workflows eliminate duplication — define once, call anywhere
- `needs` + `outputs` chain jobs and pass data between them
- `environment: production` gives a free manual approval gate
- `$GITHUB_STEP_SUMMARY` renders markdown directly in Actions UI
- `workflow_dispatch` lets you test scheduled workflows manually
- Always check file encoding on Windows — PowerShell creates UTF-16 by default
