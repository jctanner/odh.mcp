# odh-mcp

MCP server for orchestrating the full RHOAI stack — workbenches, model serving, InferenceServices, S3 connections, and more — through a single unified interface.

Agents connect to it for interactive exploration; scripts use the same classes for CI automation. Works with any Jupyter server and Kubernetes cluster accessible via URL — on OpenShift, Kubernetes, or locally.

## Why not jupyter-mcp-server?

Datalayer's [jupyter-mcp-server](https://github.com/datalayer/jupyter-mcp-server) is a full-featured MCP server for Jupyter notebooks, but it's scoped to notebooks only. This project exists because we need a single MCP server that covers the full RHOAI platform: workbenches, model serving (ServingRuntimes, InferenceServices), S3 data connections, and eventually pipelines and model registry.

The notebook tools here (Phase 1) are intentionally simpler — no three-backend abstraction (YDoc/file/WebSocket), no Jupyter extension mode — just a thin client pointed at a remote workbench URL. The real value is Phase 2+, where agents can deploy models, check InferenceService status, create connections, and run inference tests all through one server.

## Architecture

```
Agent / Script
    │
    ▼ (MCP over stdio)
┌─────────────────────────────────┐
│  odh-mcp server                 │
│  ┌───────────┐ ┌──────────────┐ │
│  │ Notebook   │ │ Kubernetes   │ │
│  │ Tools      │ │ Tools (TBD)  │ │
│  └─────┬─────┘ └──────┬───────┘ │
│        │               │         │
│  jupyter-server-client  │         │
│  jupyter-kernel-client  kubectl   │
└────────┼───────────────┼─────────┘
         │               │
         ▼               ▼
   Workbench Pod    K8s API Server
   (Jupyter)
```

## Tools

### Notebook tools (Phase 1 — implemented)

| Tool | Description |
|------|-------------|
| `list_notebooks` | List .ipynb files on the workbench |
| `list_files` | List files and directories on the workbench |
| `read_notebook` | Read all cells (source + outputs) from a notebook |
| `read_cell` | Read a single cell's source and outputs |
| `start_kernel` | Start a Jupyter kernel on the workbench |
| `execute_cell` | Execute one cell by index, return outputs |
| `execute_notebook` | Execute all code cells sequentially |
| `execute_code` | Execute arbitrary Python code in the kernel |
| `shutdown_kernel` | Stop the active kernel |

### Kubernetes / RHOAI tools (Phase 2 — planned)

| Tool | Description |
|------|-------------|
| `get_pod_status` | Check pod readiness in namespace |
| `get_inferenceservice` | Get ISVC status and conditions |
| `deploy_model` | Create ServingRuntime + InferenceService |
| `test_inference` | Send prediction request to model endpoint |
| `create_connection` | Create S3 data connection secret |

## Install

```bash
pip install -e .
```

## Usage

### As an MCP server (Claude Code, agents)

```bash
odh-mcp --workbench-url https://rh-ai.apps.ocp-sim.test/notebook/fraud-detection/fraud-detection
```

Add to your MCP config to use with Claude Code or other MCP clients.

### As a library

```python
import asyncio
from odh_mcp.notebook import NotebookRunner

async def main():
    runner = NotebookRunner("https://rh-ai.apps.ocp-sim.test/notebook/fraud-detection/fraud-detection")
    runner.start_kernel()
    results = await runner.execute_notebook("fraud-detection/1_experiment_train.ipynb")
    for r in results:
        print(r.summary())
    runner.shutdown()

asyncio.run(main())
```

## Configuration

CLI flags override environment variables:

| Flag | Env var | Description |
|------|---------|-------------|
| `--workbench-url` | `ODH_MCP_WORKBENCH_URL` | Jupyter workbench URL (required) |
| `--workbench-token` | `ODH_MCP_WORKBENCH_TOKEN` | Jupyter auth token |
| `--namespace` | `ODH_MCP_NAMESPACE` | Kubernetes namespace |
| `--kubeconfig` | `KUBECONFIG` | Path to kubeconfig |
| `--no-verify-ssl` | `ODH_MCP_VERIFY_SSL=false` | Disable SSL verification |
