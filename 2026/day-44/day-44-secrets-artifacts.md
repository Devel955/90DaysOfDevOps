# Day 44 – Secrets, Artifacts & Running Real Tests in CI

## 📌 Overview
In this day, I worked on making CI pipelines more practical and closer to real-world usage using GitHub Actions.
---
## 🔐 Secrets Management

- Created and used GitHub Secrets:
  - `MY_SECRET_MESSAGE`
  - `DOCKER_USERNAME`
  - `DOCKER_TOKEN`
- Accessed secrets securely inside workflows using:
  ```yaml
  ${{ secrets.SECRET_NAME }}

  Key Learning
Secrets should never be printed in logs
GitHub automatically masks sensitive values (***)
Helps prevent accidental data leaks

📦 Artifacts Handling
Generated files during workflow execution
Uploaded artifacts using:
uses: actions/upload-artifact@v4

Downloaded artifacts in another job using:
uses: actions/download-artifact@v4

Use Cases
Sharing data between jobs
Storing logs and reports
Passing build outputs to deployment stages

🔁 Multi-Job Workflow
Implemented workflows with multiple jobs
Used needs: to define job dependencies
Example
needs: generate-artifact

🧪 Running Real Tests in CI
Added a Python script and executed it in CI
Pipeline fails if the script exits with a non-zero status
Key Learning
CI pipelines should automatically detect failures

Helps catch bugs early in development
⚡ Caching for Performance
Used caching to speed up dependency installation:
uses: actions/cache@v4
Cached Path
~/.cache/pip
Benefits
Faster workflow execution
Reduced dependency installation time

📊 Final Outcome
Successfully implemented:
Secrets management
Artifact upload/download
Multi-job workflows
Real test execution
Dependency caching

🚀 Key Takeaways
CI is not just automation, it is validation
Security is critical when working with pipelines
Artifacts enable communication between jobs
Caching improves efficiency in large pipelines
🧠 What I Learned
This day helped me understand how real-world CI pipelines are designed with:
Security (Secrets)
Data flow (Artifacts)
Validation (Tests)
Performance (Caching)
