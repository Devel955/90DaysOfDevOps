# Day 41 – Triggers & Matrix Builds

## Challenge Tasks

---

### Task 1: Trigger on Pull Request

1. Create `.github/workflows/pr-check.yml`
2. Trigger it only when a pull request is opened or updated against `main`
3. Add a step that prints: `PR check running for branch: <branch name>`
4. Create a new branch, push a commit, and open a PR
5. Watch the workflow run automatically

**Verify: Does it show up on the PR page?**
> ✅ Yes, it shows up on the PR page.

**PR Check Workflow**

```yaml
name: PR Check

on:
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize

jobs:
  pr-check:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print branch name
        run: |
          echo "PR check running for branch: ${{ github.head_ref }}"
```

---

### Task 2: Scheduled Trigger

1. Add a `schedule:` trigger to any workflow using cron syntax
2. Set it to run every day at midnight UTC
3. What is the cron expression for every Monday at 9 AM?

**Schedule Trigger**

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

> ✅ Cron expression for every Monday at 9 AM is `0 9 * * 1`

---

### Task 3: Manual Trigger

1. Create `.github/workflows/manual.yml` with a `workflow_dispatch:` trigger
2. Add an input that asks for an `environment` name (staging/production)
3. Print the input value in a step
4. Go to the Actions tab → find the workflow → click Run workflow

**Verify: Can you trigger it manually and see your input printed?**
> ✅ Yes, I can see the input value.

**Manual Trigger**

```yaml
name: Manual Trigger

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Choose environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

  schedule:
    - cron: '0 0 * * *'

jobs:
  manual-job:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print environment input
        run: |
          echo "Deploying to environment: ${{ github.event.inputs.environment }}"

      - name: Scheduled run notice
        if: github.event_name == 'schedule'
        run: |
          echo "This is a scheduled run at midnight UTC"
```

---

### Task 4: Matrix Builds

Create `.github/workflows/matrix.yml` that:
1. Uses a matrix strategy to run the same job across Python versions: `3.10`, `3.11`, `3.12`
2. Each job installs Python and prints the version
3. Watch all 3 run in parallel

**Then extend the matrix to also include 2 operating systems — how many total jobs run now?**
> ✅ Total **6 jobs** run (3 Python versions × 2 OS).

**Matrix Builds**

```yaml
name: Matrix Build

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print Python version
        run: python --version
```

**Extend Matrix (with OS)**

```yaml
name: Matrix Build (Extended)

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print Python version
        run: python --version

      - name: Confirm OS and Python combo
        run: |
          echo "Running on OS: ${{ matrix.os }}"
          echo "Python version: ${{ matrix.python-version }}"
```

---

### Task 5: Exclude & Fail-Fast

1. In your matrix, exclude one specific combination (e.g., Python 3.10 on Windows)
2. Set `fail-fast: false` — trigger a failure in one job and observe what happens to the rest

**Exclude & Fail-Fast**

```yaml
name: Matrix Build

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]
        exclude:
          - os: windows-latest
            python-version: "3.10"

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print Python version
        run: python --version

      - name: Simulate failure on Python 3.11
        if: matrix.python-version == '3.11'
        run: exit 1
```

**Observation (`fail-fast: false`):**

- In the workflow, a failure was triggered for Python 3.11 using `exit 1`.
- As shown in the screenshot, the jobs (`windows-latest, 3.11`) and (`ubuntu-latest, 3.11`) **failed**, but the other jobs continued running and completed successfully.
- This shows that `fail-fast: false` allows all matrix jobs to run even if some fail.

**`fail-fast: true` (default):** If one job fails, the remaining matrix jobs are **cancelled immediately**.

**`fail-fast: false`:** If one job fails, the other jobs **continue running** until completion.

---
