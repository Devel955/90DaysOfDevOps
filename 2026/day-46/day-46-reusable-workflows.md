# Day 46 – Reusable Workflows & Composite Actions

Today I learned how to create reusable workflows and composite actions in GitHub Actions.

This helps avoid repetition and makes CI/CD pipelines modular and reusable.

---

## 🔹 Task 1: Understanding workflow_call

### What is a reusable workflow?
A reusable workflow is a GitHub Actions workflow that can be called by another workflow using `workflow_call`.  
It allows reuse of entire workflows across repositories.

### What is workflow_call?
`workflow_call` is a trigger that allows one workflow to be invoked by another workflow.

### Difference between reusable workflow and action
- Reusable workflow = full workflow (can contain jobs)
- Action = step-level reuse (used inside jobs)

### Where must reusable workflows live?
Reusable workflows must be inside:
.github/workflows/


---

## 🔹 Task 2: Reusable Workflow

### File: `.github/workflows/reusable-build.yml`

```yaml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      app_name:
        required: true
        type: string
      environment:
        required: true
        default: "staging"
        type: string
    secrets:
      docker_token:
        required: true
    outputs:
      build_version:
        description: "Generated build version"
        value: ${{ jobs.build.outputs.build_version }}

jobs:
  build:
    runs-on: ubuntu-latest

    outputs:
      build_version: ${{ steps.version.outputs.build_version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print build info
        run: echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Check secret
        run: echo "Docker token is set: ${{ secrets.docker_token != '' }}"

      - name: Generate version
        id: version
        run: echo "build_version=v1.0-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

🔹 Task 3: Caller Workflow

File: .github/workflows/call-build.yml
name: Call Reusable Build

on:
  push:
    branches:
      - main

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      app_name: "my-web-app"
      environment: "production"
    secrets:
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  print-version:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Print version
        run: echo "Build version: ${{ needs.build.outputs.build_version }}"

🔹 Task 4: Outputs

Generated version: v1.0-<short-sha>
Passed from reusable workflow → caller workflow
Accessed using:
needs.build.outputs.build_version

🔹 Task 5: Composite Action

File: .github/actions/setup-and-greet/action.yml
name: "Setup and Greet"
description: "A composite action that prints greeting"

inputs:
  name:
    description: "User name"
    required: true
  language:
    description: "Language"
    required: false
    default: "en"

outputs:
  greeted:
    description: "Greeting status"
    value: ${{ steps.greet.outputs.greeted }}

runs:
  using: "composite"
  steps:
    - name: Print greeting
      id: greet
      shell: bash
      run: |
        if [ "${{ inputs.language }}" = "en" ]; then
          echo "Hello, ${{ inputs.name }}!"
        else
          echo "Namaste, ${{ inputs.name }}!"
        fi

        echo "Date: $(date)"
        echo "Runner OS: $RUNNER_OS"

        echo "greeted=true" >> $GITHUB_OUTPUT


🔹 Composite Action Workflow
File: .github/workflows/composite-demo.yml
name: Composite Action Demo

on:
  workflow_dispatch:

jobs:
  run-composite:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run composite action
        id: greet_step
        uses: ./.github/actions/setup-and-greet
        with:
          name: "Anand"
          language: "en"

      - name: Print output
        run: |
          echo "Greeted: ${{ steps.greet_step.outputs.greeted }}"


🔹 Task 6: Comparison Table
| Reusable Workflow             | Composite Action            |
| ----------------------------- | --------------------------- |
| Triggered by `workflow_call`  | Used via `uses:` in a step  |
| Can contain jobs              | Cannot contain jobs         |
| Can contain multiple steps    | Can contain multiple steps  |
| Lives in `.github/workflows/` | Lives in `.github/actions/` |
| Can accept secrets directly   | Secrets passed via workflow |
| Best for full pipeline reuse  | Best for step reuse         |


🔹 Key Learnings
Reusable workflows help reuse full CI/CD pipelines
Composite actions help reuse steps
Outputs can be passed between jobs
YAML structure and indentation is critical
workflow_call enables modular workflows


