# Day 39 – CI/CD Concepts

## Task 1: The Problem

### 1) What can go wrong?

When a team of 5 developers pushes code to the same repository and deploys manually to production, several issues can occur:

- Human errors (wrong commands, wrong files, wrong server)
- Code conflicts between developers
- Missing deployment steps
- Inconsistent environments (local vs production)
- Bugs reaching production due to lack of proper testing
- Application downtime
- Difficult rollback process
- Unreliable and inconsistent deployments

---

### 2) What does "it works on my machine" mean?

"It works on my machine" means that the code runs successfully on a developer’s local system but fails on another system or in production.

#### Why is this a real problem?

- Different environments (OS, configurations)
- Different software versions
- Missing dependencies
- Environment-specific bugs
- Lack of consistency across systems

This leads to unreliable applications and deployment failures.

---

### 3) How many times a day can a team safely deploy manually?

A team can safely deploy manually only **1–2 times per day** (sometimes even less).

#### Reasons:

- High risk of human error
- Manual testing takes time
- Coordination between team members is required
- Increased chances of breaking production with frequent deploy

## Task 2: CI vs CD

### 1) Continuous Integration (CI)

Continuous Integration is the practice of frequently integrating code changes into a shared repository, usually multiple times a day. Each change is automatically built and tested.

It helps catch bugs early such as integration issues, broken builds, and failing tests.

#### Real-world Example:
A developer pushes code to GitHub → Jenkins/GitHub Actions automatically runs tests → if tests fail, the code is rejected.

---

### 2) Continuous Delivery (CD)

Continuous Delivery is an extension of CI where code changes are automatically prepared for release to production. The deployment is ready but requires manual approval to go live.

"Delivery" means the application is always in a deployable state.

#### Real-world Example:
After passing all tests, the app is automatically deployed to a staging environment → team manually approves → then it goes to production.

---

### 3) Continuous Deployment

Continuous Deployment goes one step further than Continuous Delivery. Every change that passes all tests is automatically deployed to production without any manual approval.

This allows very fast and frequent releases.

#### Real-world Example:
A company like Netflix pushes small updates → once tests pass, changes are automatically deployed to production without human intervention.

---
## Summary
- CI → Code is integrated, built, and tested frequently
- Continuous Delivery → Code is ready for deployment (manual approval needed)
- Continuous Deployment → Code is automatically deployed to production

## Task 3: Pipeline Anatomy

A CI/CD pipeline is made up of multiple components. Each part has a specific role in automating the software delivery process.

---

### 1) Trigger

A Trigger is the event that starts the pipeline.

It can be a code push, pull request, manual action, or a scheduled event.

#### Example:
A developer pushes code to GitHub → pipeline automatically starts.

---

### 2) Stage

A Stage is a logical phase in the pipeline that groups related jobs.

Common stages include build, test, and deploy.

#### Example:
Stages: Build → Test → Deploy

---

### 3) Job

A Job is a unit of work inside a stage.

Each job runs a set of steps on a runner.

#### Example:
In the "Test" stage, one job runs unit tests, another runs integration tests.

---

### 4) Step

A Step is a single command or action inside a job.

Steps are executed in sequence within a job.

#### Example:
- Install dependencies  
- Run tests  
- Build application  

---

### 5) Runner

A Runner is the machine (server/VM/container) that executes the jobs.

It can be self-hosted or provided by platforms like GitHub Actions.

#### Example:
GitHub-hosted runner runs your pipeline on a virtual machine.

---

### 6) Artifact

An Artifact is the output produced by a job.

It is stored and passed to later stages or used for deployment.

#### Example:
A compiled application (.jar, .zip, Docker image) generated in the build stage.
---
## Summary
- Trigger → Starts the pipeline  
- Stage → Logical phase (build, test, deploy)  
- Job → Task inside a stage  
- Step → Individual command inside a job  
- Runner → Executes the job  
- Artifact → Output of the pipeline

GitHub Markdown (Task 4)
  [ Developer ]
     |
     v
[ Push Code to GitHub ]
     |
     v
[ Trigger Pipeline ]

     |
     v
========================
  Stage 1: Build
========================
- Install dependencies  
- Build app  

     |
     v
========================
  Stage 2: Test
========================
- Run unit tests  
- Check errors  

     |
     v
========================
  Stage 3: Docker
========================
- Build Docker image  
- Tag image  

     |
     v
========================
 Stage 4: Deploy (Staging)
========================
- Push image  
- Deploy to server  

     |
     v
[ App Running on Staging 🚀 ]

## Task 5: Explore in the Wild

I explored the **React** GitHub repository and checked one workflow file inside `.github/workflows/`.

**Repository:** `facebook/react`  
**Workflow file:** `.github/workflows/compiler_typescript.yml`

### 1) What triggers it?

This workflow is triggered by:

- **push** events on the **main** branch
- **pull_request** events when changes affect:
  - `compiler/**`
  - `.github/workflows/compiler_typescript.yml`

### 2) How many jobs does it have?

It has **4 jobs**:

- `discover_yarn_workspaces`
- `lint`
- `jest`
- `test`

> Note: The `test` job uses a **matrix**, so it can run the same test job for multiple workspaces.

### 3) What does it do? (Best guess)

This workflow appears to be a **CI workflow** for React’s compiler-related code.

It likely does the following:

- checks out the code
- sets up Node.js
- restores/install dependencies
- finds compiler workspaces
- runs lint checks
- runs Jest tests
- runs workspace-level tests
### Conclusion
This workflow helps the React team automatically verify compiler-related code changes before merging them. If something breaks, the workflow fails early, which is exactly how CI/CD should help developers.
