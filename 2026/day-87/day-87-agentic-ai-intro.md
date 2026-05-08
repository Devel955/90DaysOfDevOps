# Day 87 — Introduction to Agentic AI for DevOps

## What Are AI Agents and How They Differ from Chatbots

A **chatbot** takes a user message and returns a text reply. It has no memory of state, no access to tools, and cannot take actions in the real world.

An **AI agent** is fundamentally different:

- It wraps an LLM with **tools** — Python functions that can run shell commands, call APIs, read files, or interact with any CLI
- The LLM **decides** which tool to call, with what arguments, and in what order — based on the user's question
- It operates in a **loop**: reason → act → observe → reason again — until it has a complete answer
- It can **autonomously** diagnose and fix problems without the user specifying which commands to run

For DevOps, this is transformative. Instead of asking "what does this error mean?", you can ask "why is my pod crashing?" and the agent figures out which `kubectl` commands to run, reads the output, and explains the root cause in plain English.

---

## The ReAct Pattern (Reason + Act + Observe)

ReAct is the core reasoning loop that powers modern AI agents. It stands for **Re**ason + **Act**. The agent alternates between thinking and doing until it reaches an answer.

### Example: Diagnosing broken-app

```
User: "Why is broken-app crashing?"

Agent THINKS: I should check which containers are running
Agent ACTS:   calls list_containers()
Agent OBSERVES: broken-app is in "Restarting" state

Agent THINKS: I should check the logs to understand why it's restarting
Agent ACTS:   calls get_logs("broken-app")
Agent OBSERVES: "app starting..." then "exit code 1" after 2 seconds

Agent THINKS: The container exits immediately after starting — this is a command failure
Agent ACTS:   calls inspect_container("broken-app")
Agent OBSERVES: exit code = 1, restart count = 12

Agent ANSWERS: "The container crashes because the entrypoint command exits with
               code 1 approximately 2 seconds after starting. This is typically
               caused by a missing dependency, incorrect command, or a runtime error.
               Check the Dockerfile CMD/ENTRYPOINT and verify all required files exist."
```

The key insight: **the LLM decided to check logs and inspect the container on its own**. You never told it which tools to use — it reasoned about the problem and chose the right sequence.

### Why ReAct Works

| Step | What Happens |
|------|-------------|
| **Reason** | LLM reads the question and tool docstrings, decides what information it needs |
| **Act** | LLM calls a tool with specific arguments |
| **Observe** | Tool output (stdout/stderr) is returned as text to the LLM |
| **Loop** | LLM reads the observation and decides if it has enough info or needs another tool call |
| **Answer** | When the LLM has enough information, it produces the final response |

---

## Environment Setup

### Prerequisites

```bash
# Install Ollama (local LLM runtime — no API keys needed)
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh
```

### Pull the Model and Set Up Python

```bash
# Start Ollama and pull Gemma 4
ollama serve &
ollama pull gemma4

# Verify
ollama list
# Should show: gemma4

# Clone the reference repo
git clone https://github.com/TrainWithShubham/agentic-ai-for-devops.git
cd agentic-ai-for-devops

# Create Python environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Verify Setup Output

```
python3 module-0/verify_setup.py

  [PASS] Python 3.10+
  [PASS] Docker
  [PASS] kubectl
  [PASS] Kind
  [PASS] Ollama + gemma4

  5/5 — you're ready for Day 1!
```

### Dependencies Installed

| Package | Purpose |
|---------|---------|
| `ollama` | Python client for the local Ollama LLM runtime |
| `langchain` | Agent framework — tools, prompts, chains |
| `langchain-ollama` | LangChain integration with Ollama models |
| `langgraph` | Graph-based execution engine powering `create_react_agent` |
| `fastmcp` | Model Context Protocol server framework (used on Day 88) |
| `langchain-mcp-adapters` | Bridges MCP tools into LangChain |

---

## Module 1: Docker Error Explainer

The simplest possible LLM usage — no agents, no tools. Paste a Docker error, get a plain-English explanation with fix commands.

### How It Works

```python
import ollama

SYSTEM_PROMPT = """You are a Docker expert. When given a Docker error, explain:
1. What went wrong (plain English)
2. Most likely cause
3. How to fix it (with commands)
Keep it short."""

response = ollama.chat(
    model="gemma4",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": error},
    ],
    options={"temperature": 0.3},
)
```

### Key Concepts

**System Prompt** — Sets the LLM's persona and output format. It tells the model:
- What role to play ("Docker expert")
- What structure to follow (3-point format)
- How verbose to be ("Keep it short")

Without a system prompt, the LLM responds generically. With a well-crafted system prompt, it produces focused, actionable output every time.

**Temperature: 0.3** — Controls randomness in the LLM's output:
- `0.0` = fully deterministic (same input always produces same output)
- `0.3` = mostly deterministic with slight variation (good for technical answers)
- `1.0+` = creative/varied (good for writing, brainstorming)

For DevOps troubleshooting, low temperature is preferred — you want consistent, reliable answers, not creative interpretations.

### Example Run

```
$ python3 module-1/explainer.py

Paste Docker error: docker: Error response from daemon: Conflict. The container
name "/myapp" is already in use.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EXPLANATION:
1. What went wrong: A container named "myapp" already exists on your system,
   even if it's stopped.

2. Most likely cause: You previously ran `docker run --name myapp ...` and the
   container was not removed after it stopped.

3. How to fix it:
   # Remove the existing container
   docker rm myapp

   # Or force-remove if running
   docker rm -f myapp

   # Then re-run your command
   docker run --name myapp ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Effect of System Prompt on Response Quality

| System Prompt | Response Quality |
|---------------|-----------------|
| No system prompt | Generic explanation, no structure, may be verbose |
| Vague prompt ("you are helpful") | Better but inconsistent format |
| Specific prompt with output structure | Focused, actionable, consistently formatted |
| Overly long/complex prompt | Model may ignore parts or lose focus |

**Lesson**: The system prompt is the most important tuning lever for LLM quality. Specific instructions with a defined output structure produce the best results for technical tasks.

---

## Module 2: Docker Troubleshooter Agent

A real AI agent that autonomously uses Docker CLI tools to diagnose container issues.

### Setup: Create the Broken Container

```bash
docker run -d --name broken-app nginx:alpine sh -c "echo 'app starting...' && sleep 2 && exit 1"
```

This container simulates a real crash: starts, prints a startup message, waits 2 seconds, then exits with code 1. Docker keeps restarting it — equivalent to `CrashLoopBackOff` in Kubernetes.

### The Three Tools

```python
from langchain.tools import tool
import subprocess

@tool
def list_containers() -> str:
    """List all Docker containers (running and stopped)."""
    result = subprocess.run(["docker", "ps", "-a"], capture_output=True, text=True)
    return result.stdout or result.stderr

@tool
def get_logs(container_name: str) -> str:
    """Get the last 50 lines of logs from a Docker container."""
    result = subprocess.run(
        ["docker", "logs", "--tail", "50", container_name],
        capture_output=True, text=True,
    )
    return result.stdout + result.stderr

@tool
def inspect_container(container_name: str) -> str:
    """Get detailed info about a Docker container (state, config, network)."""
    result = subprocess.run(
        ["docker", "inspect", container_name],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr
```

**How tools work:**
- `@tool` decorator registers the function with LangChain's agent framework
- The **docstring** is what the LLM reads to decide *when* to use this tool — it must be clear and accurate
- `subprocess.run` executes the actual CLI command
- Returns `stdout`/`stderr` as a plain string for the LLM to parse

### Creating the Agent

```python
from langchain_ollama import ChatOllama
from langgraph.prebuilt import create_react_agent

llm = ChatOllama(model="gemma4", temperature=0)
tools = [list_containers, get_logs, inspect_container]
agent = create_react_agent(llm, tools)
```

`create_react_agent` handles the entire ReAct loop automatically — you don't write any reasoning logic.

### Example Agent Run

```
$ python3 module-2/agent.py

> Why is broken-app crashing?

[Calling list_containers...]
CONTAINER ID   NAME         STATUS
a3f2b1c4d5e6   broken-app   Restarting (1) 3 seconds ago

[Calling get_logs(broken-app)...]
app starting...

[Calling inspect_container(broken-app)...]
"ExitCode": 1,
"RestartCount": 8,
"FinishedAt": "2026-05-07T10:23:15Z"

ANSWER: The container "broken-app" is crashing because its entrypoint command
exits with code 1 approximately 2 seconds after starting. The log shows only
"app starting..." before the exit, indicating the failure occurs in the startup
script itself. Exit code 1 is a general error — the command (sh -c "... exit 1")
is deliberately failing. To fix: check the Dockerfile's CMD/ENTRYPOINT and verify
all required dependencies and files are present inside the container.
```

### Cleanup

```bash
docker rm -f broken-app
```

---

## Agent Architecture Diagram

```
[User Question: "Why is broken-app crashing?"]
           │
           ▼
   ┌───────────────┐
   │  Gemma 4 LLM  │  ← Reads question + tool docstrings
   │  via Ollama   │  ← Reasons: "I need to check containers first"
   └───────┬───────┘
           │ ReAct: Tool Selection
           ▼
   ┌───────────────────────────────────┐
   │         Tool Dispatcher           │
   └─────┬──────────┬──────────┬───────┘
         │          │          │
         ▼          ▼          ▼
  list_containers  get_logs  inspect_container
  docker ps -a    docker     docker inspect
                  logs --    <container>
                  tail 50

         │          │          │
         └──────────┴──────────┘
                    │
                    ▼ Tool Output (text)
   ┌───────────────────┐
   │    LLM reads      │
   │    observation,   │
   │    reasons again  │
   └────────┬──────────┘
            │ (repeat until answer is ready)
            ▼
   [Final Answer to User]
```

---

## Extended Tool: list_images

Added to `module-2/agent.py`:

```python
@tool
def list_images() -> str:
    """List all Docker images on this machine with their sizes."""
    result = subprocess.run(["docker", "images"], capture_output=True, text=True)
    return result.stdout or result.stderr
```

```python
tools = [list_containers, get_logs, inspect_container, list_images]
```

**Agent response to "What images do I have and how much space are they using?":**

```
[Calling list_images...]
REPOSITORY    TAG       IMAGE ID       SIZE
nginx         alpine    3f8a4339bbe2   43.3MB
python        3.11      a72db524e878   1.02GB
node          20        c7f1a3d6e291   1.13GB

ANSWER: You have 3 Docker images using approximately 2.19GB of disk space:
- nginx:alpine (43MB) — lightweight web server
- python:3.11 (1.02GB) — Python runtime
- node:20 (1.13GB) — Node.js runtime

To free space, remove unused images with: docker image prune
```

The agent called `list_images` automatically because the question was about images and space — it matched the tool's docstring.

---

## Extended Tool: restart_container (with Safety Discussion)

```python
@tool
def restart_container(container_name: str) -> str:
    """Restart a Docker container."""
    result = subprocess.run(["docker", "restart", container_name], capture_output=True, text=True)
    return result.stdout or result.stderr
```

**Safety implications:**

This tool can restart *any* container by name. In production environments, this requires guardrails:

| Risk | Mitigation |
|------|-----------|
| Agent restarts wrong container | Allowlist of permitted container names |
| Restart causes downtime | Require human confirmation before acting |
| Loop: agent restarts crashing container forever | Max restart count + detection logic |
| Unauthorized access | Scope agent permissions (read-only mode by default) |

Day 89 covers production-grade guardrails including human-in-the-loop confirmation.

---

## Key Concepts Summary

### System Prompt
The system prompt defines the agent's persona, capabilities, and output format. It is the most effective way to tune LLM behavior without changing the model or code. For technical agents, a specific, structured system prompt dramatically improves output quality and consistency.

### Temperature
Controls output randomness. For DevOps agents: use `temperature=0` for deterministic, reproducible answers. For creative tasks (documentation, explanations): `0.3–0.7` adds helpful variation.

### The @tool Docstring
The docstring is the LLM's only source of information about what a tool does. It must answer: *when should I use this tool?* and *what does it return?* Bad docstrings lead to wrong tool selection or the agent getting stuck in reasoning loops.

### Why This Pattern is Powerful
The architecture is domain-agnostic. The same ReAct loop works for:
- Docker → `docker ps`, `docker logs`, `docker inspect`
- Kubernetes → `kubectl get pods`, `kubectl describe`, `kubectl logs`
- Terraform → `terraform plan`, `terraform state list`
- AWS → `aws ec2 describe-instances`, `aws cloudwatch get-metric-data`

Any CLI tool can become an agent tool. Any DevOps workflow can be automated this way.

---

## What's Next

| Day | Topic |
|-----|-------|
| Day 88 | Kubernetes tools + MCP (Model Context Protocol) for standardized tool exposure |
| Day 89 | Production-grade agent: automatic pod healing with guardrails |
| Day 90 | Full pipeline: GitOps + AI agent for end-to-end automated remediation |
