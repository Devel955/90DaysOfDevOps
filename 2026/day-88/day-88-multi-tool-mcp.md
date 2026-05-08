# Day 88 — Multi-Tool Agents, MCP, and CI/CD Analyzer

## Overview

Today's challenge extended the Docker agent from Day 87 into a full multi-domain DevOps agent. The work covers three major areas:

1. A multi-tool agent handling both Docker and Kubernetes (6 tools, 2 domains)
2. An MCP (Model Context Protocol) server exposing Kubernetes tools to any AI client
3. A CI/CD Failure Analyzer that diagnoses broken GitHub Actions pipelines

---

## Task 1: Multi-Tool DevOps Agent Architecture

### The 6-Tool Setup

The agent now operates across two infrastructure domains from a single conversation:

| Domain     | Tool                | What It Does                              |
|------------|---------------------|-------------------------------------------|
| Docker     | `list_containers`   | Runs `docker ps -a` to show all containers |
| Docker     | `get_logs`          | Fetches `docker logs <name>`              |
| Docker     | `inspect_container` | Runs `docker inspect <name>`              |
| Kubernetes | `list_pods`         | Runs `kubectl get pods -n <namespace>`    |
| Kubernetes | `describe_pod`      | Runs `kubectl describe pod <name>`        |
| Kubernetes | `get_events`        | Runs `kubectl get events --sort-by=.lastTimestamp` |

### Broken Resources for Testing

```bash
# Kind cluster with a crashing pod
kind create cluster --name devops-demo
kubectl apply -f module-3/broken_pod.yaml

# broken_pod.yaml — pod that exits with code 1
# command: ["sh", "-c", "echo 'app starting...' && sleep 2 && exit 1"]

# Broken Docker container
docker run -d --name broken-container nginx:alpine \
  sh -c "echo 'container starting...' && sleep 2 && exit 1"
```

### Agent Diagnosis (Sample Interaction)

```
> What's broken across Docker and Kubernetes?

[Agent calls list_containers] → finds broken-container in Exited state
[Agent calls get_logs] → sees "container starting..." then exit
[Agent calls list_pods] → finds broken-pod in CrashLoopBackOff
[Agent calls describe_pod] → sees exit code 1, back-off restarting

Both resources are crashing intentionally:
- Docker: broken-container exits after 2 seconds (exit 1)
- Kubernetes: broken-pod crashes immediately (exit 1), triggering CrashLoopBackOff
Root cause: the container command explicitly exits with a non-zero code.
```

### The ReAct Pattern in Action

The agent doesn't need separate logic for Docker vs Kubernetes. The LLM reads each tool's docstring, reasons about what information it needs, and selects the appropriate tool dynamically. Ask a Docker question — it picks Docker tools. Ask about pods — it switches to `kubectl` tools. Ask about both — it uses all six.

---

## Task 2: Model Context Protocol (MCP) — Explained

### What Is MCP?

MCP (Model Context Protocol) is an open standard created by Anthropic for connecting AI models to external tools and data sources. Instead of writing tools directly inside your agent code, you expose them as a discoverable service via MCP. Any compatible AI client can then find and call those tools.

### Why MCP Matters for DevOps

| Without MCP | With MCP |
|---|---|
| Tools are locked to one framework (e.g., LangChain) | Tools work with any MCP-compatible client |
| Every AI re-implements Docker/K8s integrations | Write once, reuse everywhere |
| Tool access is tied to your agent's codebase | Tools exposed as a discoverable, standalone service |
| Hard to share tools across teams or clients | Tools are versioned and centralised |

### MCP-Compatible Clients

- Claude Desktop
- VS Code (GitHub Copilot)
- Cursor
- Claude Code CLI
- Any LangChain agent via `langchain-mcp-adapters`

### Architecture

```
┌─────────────────────────┐         ┌────────────────────────┐
│       MCP Server        │         │      MCP Clients        │
│  mcp_server.py          │◄───────►│  Claude Desktop         │
│                         │         │  VS Code Copilot        │
│  • list_pods()          │  stdio  │  Your Python Agent      │
│  • describe_pod()       │  / HTTP │  Claude Code CLI        │
│  • get_events()         │         │  Any MCP client         │
└─────────────────────────┘         └────────────────────────┘
```

---

## Task 3: MCP Server vs Hardcoded LangChain Tools

### Hardcoded LangChain Tool (agent.py)

```python
from langchain.tools import tool

@tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(["kubectl", "get", "pods", "-n", namespace],
                            capture_output=True, text=True)
    return result.stdout or result.stderr
```

- Defined *inside* the agent script
- Only available to that specific agent
- Changing tools requires modifying the agent code

### MCP Server Tool (mcp_server.py)

```python
from fastmcp import FastMCP

mcp = FastMCP("Kubernetes Tools")

@mcp.tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(["kubectl", "get", "pods", "-n", namespace],
                            capture_output=True, text=True)
    return result.stdout or result.stderr

if __name__ == "__main__":
    mcp.run()  # Starts the MCP server (stdio by default)
```

Key differences:
- `@mcp.tool` registers with the MCP server, not LangChain
- `mcp.run()` exposes tools as a service
- Any MCP client can discover and call them at runtime

### MCP Client Agent (agent_with_mcp.py)

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async def main():
    client = MultiServerMCPClient({
        "kubernetes-mcp": {
            "transport": "stdio",
            "command": "python",
            "args": ["mcp_server.py"]
        }
    })

    tools = await client.get_tools()   # Discovers tools dynamically from MCP
    agent = create_agent(llm, tools)   # Same ReAct agent — tools sourced from MCP
```

The agent does **not** define tools locally. It discovers them at runtime from the MCP server. This is the key architectural shift.

### Claude Desktop Integration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "kubernetes-tools": {
      "command": "python3",
      "args": ["/full/path/to/agentic-ai-for-devops/module-3/mcp_server.py"]
    }
  }
}
```

After restarting Claude Desktop, asking "List the pods in my cluster" triggers the MCP server's `list_pods()` tool directly from the Claude UI.

---

## Task 4: CI/CD Failure Analyzer

### Tools

```python
@tool
def list_workflow_runs(status: str = "failure") -> str:
    """List recent GitHub Actions workflow runs. Use status='failure' for failed runs."""
    result = subprocess.run(
        ["gh", "run", "list", "--status", status, "--limit", "5"],
        capture_output=True, text=True
    )
    return result.stdout or result.stderr

@tool
def get_failed_logs(run_id: str) -> str:
    """Get the failed step logs from a GitHub Actions run. Pass the run ID."""
    result = subprocess.run(
        ["gh", "run", "view", run_id, "--log-failed"],
        capture_output=True, text=True
    )
    output = result.stdout + result.stderr
    if len(output) > 5000:
        output = output[:5000] + "\n\n[...truncated, showing first 5000 chars]"
    return output

@tool
def get_workflow_file(workflow_name: str) -> str:
    """Read a GitHub Actions workflow YAML file. Pass the filename like 'ci.yml'."""
    path = pathlib.Path(f".github/workflows/{workflow_name}")
    return path.read_text() if path.exists() else f"File not found: {path}"
```

### Why Log Truncation Matters

CI/CD logs can easily exceed 100KB. LLMs have token limits and work best with focused input. Truncating to 5000 characters keeps the most critical output (the failed step) within context while avoiding prompt overflow. This is a practical engineering constraint, not a workaround.

### Sample Interaction

```
> What failed in my last CI run?

[Agent calls list_workflow_runs] → finds run ID 1234567890, status: failure
[Agent calls get_failed_logs(run_id="1234567890")]
→ "npm test — No such file or directory: package.json"

Root cause: The workflow runs `npm test` but there is no package.json
in the repository. The step assumes a Node.js project structure that
doesn't exist. Fix: either add a package.json or remove the npm test step.
```

### Broken Workflow for Testing

```yaml
# .github/workflows/broken-ci.yml
name: Broken CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test    # Fails — no package.json present
```

Push, let it fail, then ask: *"Why did broken-ci fail?"* — the agent will fetch the logs, identify the missing `package.json`, and explain the fix.

---

## Task 5: Custom Tool — Log Searcher

### The Tool

```python
@tool
def search_logs(keyword: str, namespace: str = "default") -> str:
    """Search for a keyword in the logs of all pods in a namespace.
    Use this when investigating errors that could be spread across multiple pods."""
    pods = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace, "-o", "name"],
        capture_output=True, text=True
    )
    results = []
    for pod in pods.stdout.strip().split("\n"):
        if not pod:
            continue
        logs = subprocess.run(
            ["kubectl", "logs", pod, "-n", namespace, "--tail=100"],
            capture_output=True, text=True
        )
        if keyword.lower() in logs.stdout.lower():
            results.append(f"{pod}: found '{keyword}'")
    return "\n".join(results) if results else f"No pods contain '{keyword}' in their logs"
```

### How the Agent Decided When to Use It

When asked: *"Which pods have logged an OOMKilled error?"* — the agent recognised that this requires searching across all pods rather than inspecting a single one. The docstring phrase *"spread across multiple pods"* was the signal. The agent called `search_logs(keyword="OOMKilled")` directly, without needing guidance.

This demonstrates that **docstrings are the agent's documentation** — the more specific and descriptive, the better the tool selection.

---

## Universal Tool Pattern

Any CLI command can become an agent tool. The pattern is always the same:

```python
@tool
def your_tool_name(param: str) -> str:
    """One-line description of what this tool does and WHEN to use it.
    Be specific — this is how the agent decides which tool to call."""
    result = subprocess.run(
        ["your-cli", "command", param],
        capture_output=True,
        text=True
    )
    output = result.stdout or result.stderr
    # Truncate if needed to stay within token limits
    if len(output) > 5000:
        output = output[:5000] + "\n[...truncated]"
    return output
```

Steps:
1. Identify a CLI command that provides useful diagnostic or operational output
2. Wrap it with the pattern above
3. Write a clear, specific docstring describing when to use it
4. Add it to the tools list of any ReAct agent
5. The agent handles all reasoning about when and how to call it

---

## Module Summary

| Module | Component | Tools | Pattern |
|---|---|---|---|
| 3 — agent.py | Multi-tool agent | 3 Docker + 3 K8s | LangChain ReAct |
| 3 — mcp_server.py | MCP server | 3 K8s tools via MCP | FastMCP |
| 3 — agent_with_mcp.py | MCP client agent | Tools discovered from MCP | LangChain + MCP adapter |
| 6 — ci_analyzer.py | CI/CD analyzer | 3 GitHub Actions tools | LangChain ReAct |

---

## Key Takeaways

- **One agent, many tools, one brain** — the ReAct pattern scales to any number of CLI-backed tools across any domain
- **MCP decouples tools from agents** — write once, use from Claude Desktop, VS Code, Python, or any compatible client
- **Docstrings drive tool selection** — the LLM reads them to decide which tool to call; invest time in writing them well
- **Log truncation is not optional** — CI logs can be enormous; always cap output before sending it to an LLM
- **`temperature=0` for troubleshooting** — deterministic output is essential when diagnosing infrastructure issues

---

## Cleanup

```bash
kind delete cluster --name devops-demo
docker rm -f broken-container 2>/dev/null
deactivate
```

---

*Day 88 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
