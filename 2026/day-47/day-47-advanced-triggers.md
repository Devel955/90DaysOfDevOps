# Day 47 – Advanced Triggers: PR Events, Cron Schedules & Event-Driven Pipelines

Today I explored advanced GitHub Actions triggers and learned how event-driven CI/CD pipelines work.

## Task 1: Pull Request Lifecycle Events

I created a workflow `pr-lifecycle.yml` that triggers on pull request activity types:
- opened
- synchronize
- reopened
- closed

It prints:
- event type
- PR title
- PR author
- source branch
- target branch

It also has a conditional step that runs only when the pull request is merged.

### Workflow: `.github/workflows/pr-lifecycle.yml`

```yaml
name: PR Lifecycle

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  pr-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print PR info
        run: |
          echo "Event: ${{ github.event.action }}"
          echo "PR Title: ${{ github.event.pull_request.title }}"
          echo "PR Author: ${{ github.event.pull_request.user.login }}"
          echo "Source Branch: ${{ github.head_ref }}"
          echo "Target Branch: ${{ github.base_ref }}"

Task 2: PR Validation Workflow

I created pr-checks.yml to validate pull requests targeting main.

This workflow:

fails if any file is larger than 1 MB
fails if branch name does not follow feature/*, fix/*, or docs/*
warns if the PR description is empty
Workflow: .github/workflows/pr-checks.yml

name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  file-size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check file size
        run: |
          for file in $(find . -type f); do
            size=$(stat -c%s "$file")
            if [ $size -gt 1048576 ]; then
              echo "File $file is larger than 1MB"
              exit 1
            fi
          done

  branch-name-check:
    runs-on: ubuntu-latest
    steps:
      - name: Validate branch name
        run: |
          if [[ ! "${{ github.head_ref }}" =~ ^(feature|fix|docs)/ ]]; then
            echo "Invalid branch name"
            exit 1
          fi

  pr-body-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check PR description
        run: |
          if [ -z "${{ github.event.pull_request.body }}" ]; then
            echo "Warning: PR description is empty"
          else
            echo "PR description is present"
          fi

Task 3: Scheduled Workflows (Cron Deep Dive)

I created scheduled-tasks.yml with:

30 2 * * 1 → every Monday at 2:30 AM UTC
0 */6 * * * → every 6 hours

I also added workflow_dispatch so I can test it manually.
Workflow: .github/workflows/scheduled-tasks.yml

name: Scheduled Tasks

on:
  schedule:
    - cron: '30 2 * * 1'
    - cron: '0 */6 * * *'
  workflow_dispatch:

jobs:
  scheduled-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print schedule trigger
        run: |
          echo "Triggered by: ${{ github.event.schedule }}"

      - name: Health check
        run: |
          curl -I https://google.com

Cron Expressions
Every weekday at 9 AM IST → 30 3 * * 1-5
First day of every month at midnight → 0 0 1 * *
Why scheduled workflows may be delayed or skipped

Scheduled workflows only run on the default branch. GitHub may delay or skip them on inactive repositories or during high platform load.

Task 4: Path & Branch Filters

I created workflows to:

run only when files in src/ or app/ change
skip runs when only docs or markdown files change
trigger only on main and release/* branches
Workflow: .github/workflows/smart-triggers.yml

name: Smart Triggers

on:
  push:
    branches:
      - main
      - 'release/*'
    paths:
      - 'src/**'
      - 'app/**'

jobs:
  run-job:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Triggered due to code changes in src/ or app/"

Workflow: .github/workflows/docs-ignore.yml
name: Ignore Docs Changes

on:
  push:
    branches:
      - main
      - 'release/*'
    paths-ignore:
      - '*.md'
      - 'docs/**'

jobs:
  skip-docs:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Workflow skipped for docs-only changes"

paths vs paths-ignore
paths is used when the workflow should run only for specific files or folders.
paths-ignore is used when the workflow should run for most changes but skip certain files.
Task 5: workflow_run — Chain Workflows Together

I created:

tests.yml → runs tests on every push
deploy-after-tests.yml → runs only after Run Tests completes

Deployment proceeds only if the test workflow succeeds.

Workflow: .github/workflows/tests.yml
name: Run Tests

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running tests..."

Workflow: .github/workflows/deploy-after-tests.yml
name: Deploy After Tests

on:
  workflow_run:
    workflows: ["Run Tests"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Check workflow result
        run: |
          echo "Conclusion: ${{ github.event.workflow_run.conclusion }}"

      - name: Stop if failed
        if: ${{ github.event.workflow_run.conclusion != 'success' }}
        run: |
          echo "Tests failed"
          exit 1

      - name: Deploy
        if: ${{ github.event.workflow_run.conclusion == 'success' }}
        run: |
          echo "Deploying after successful tests"

Task 6: repository_dispatch — External Event Triggers

I created external-trigger.yml to respond to external event type deploy-request.

This workflow can be triggered by:

a Slack bot sending a deploy request
a monitoring tool detecting an issue
another repository or service calling a workflow
Workflow: .github/workflows/external-trigger.yml
name: External Trigger

on:
  repository_dispatch:
    types: [deploy-request]

jobs:
  external:
    runs-on: ubuntu-latest
    steps:
      - name: Print payload
        run: |
          echo "Environment: ${{ github.event.client_payload.environment }}"

workflow_run vs workflow_call
| Feature        | workflow_run                          | workflow_call                       |
| -------------- | ------------------------------------- | ----------------------------------- |
| Trigger        | Runs after another workflow completes | Called directly by another workflow |
| Use case       | Workflow chaining                     | Reusable workflows                  |
| Inputs support | No                                    | Yes                                 |
| Best for       | Event-driven pipeline flow            | Reusing full workflows              |

In simple words:

workflow_run is used when one workflow should start after another finishes.
workflow_call is used when one workflow wants to reuse another workflow like a function.

      - name: Check if merged
        if: github.event.pull_request.merged == true
        run: echo "PR was merged"
