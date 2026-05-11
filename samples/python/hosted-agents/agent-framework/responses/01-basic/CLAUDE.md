# CLAUDE.md — Agent Framework Basic Responses Sample

## Project Overview

This is a **minimal Azure AI Foundry hosted agent** built with the [Agent Framework](https://github.com/microsoft/agent-framework) using the **Responses protocol**. It's the starting point of a progressive sample series (01→06) in the `foundry-samples` repository.

The entire agent is ~15 lines of Python: create a `FoundryChatClient`, wrap it in an `Agent`, serve it via `ResponsesHostServer`. The Responses protocol provides an OpenAI-compatible `POST /responses` endpoint with platform-managed conversation history.

## Architecture

```
User → POST /responses (port 8088) → ResponsesHostServer → Agent → FoundryChatClient → Azure AI Model
                                          ↑                    ↑
                                   Hosting SDK              Core SDK
                          (agent-framework-foundry-hosting)  (agent-framework)
```

### Core Imports

```python
from agent_framework import Agent                              # Core agent class
from agent_framework.foundry import FoundryChatClient          # Foundry model connector
from agent_framework_foundry_hosting import ResponsesHostServer # HTTP server
from azure.identity import DefaultAzureCredential              # Auth (local CLI + Foundry managed identity)
```

### Key Dependencies

- `agent-framework>=1.2.2` — Core SDK: Agent, tools, workflows, sessions
- `agent-framework-foundry-hosting` — Hosting: ResponsesHostServer, OpenTelemetry, health endpoints

## Files & Their Roles

| File | Purpose |
|------|---------|
| `main.py` | Agent logic — the only code file |
| `agent.yaml` | Runtime config: protocol (responses), resources (CPU/memory), env vars |
| `agent.manifest.yaml` | Deployment manifest: declares required Foundry resources (models) |
| `Dockerfile` | Python 3.12 slim, installs requirements, runs main.py on port 8088 |
| `requirements.txt` | Two dependencies: agent-framework, agent-framework-foundry-hosting |
| `.env.example` | Template: FOUNDRY_PROJECT_ENDPOINT, AZURE_AI_MODEL_DEPLOYMENT_NAME |
| `README.md` | Human-readable walkthrough and curl examples |

## Responses Protocol

**Endpoint:** `POST /responses` on port 8088

**Request:**
```json
{"input": "Hello!", "previous_response_id": "optional-id", "stream": true}
```

**Multi-turn:** The platform manages conversation history. Pass `previous_response_id` from the prior response to continue a conversation. The agent sets `store: False` because history storage is handled by the hosting infrastructure.

**Streaming:** When `stream: true`, returns Server-Sent Events (SSE): `response.created → response.in_progress → response.output_text.delta → response.completed`

## Environment Variables

| Variable | Source | Required |
|----------|--------|----------|
| `FOUNDRY_PROJECT_ENDPOINT` | Auto-injected by Foundry / `.env` locally | Yes |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | `agent.yaml` / `.env` locally | Yes |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Auto-injected by Foundry | No (tracing) |

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run locally (requires .env)
python main.py

# Run via Azure Developer CLI (auto-injects env vars)
azd ai agent run

# Deploy to Foundry (full setup: project + model + agent)
azd up

# Redeploy after changes
azd deploy

# Test — non-streaming
curl -X POST http://localhost:8088/responses -H "Content-Type: application/json" -d '{"input": "Hi"}'

# Test — streaming
curl -X POST http://localhost:8088/responses -H "Content-Type: application/json" -d '{"input": "Hi", "stream": true}'

# Test — multi-turn
curl -X POST http://localhost:8088/responses -H "Content-Type: application/json" \
  -d '{"input": "Tell me more", "previous_response_id": "PREVIOUS_ID"}'
```

## How to Extend This Sample

This sample is designed to be extended. The sibling samples (02→06) demonstrate each extension pattern. Here are the exact patterns:

### Add Local Tools

Use the `@tool` decorator to give the agent callable functions. The model sees the function signature and docstring and decides when to call them.

```python
from agent_framework import Agent, tool
from typing_extensions import Annotated
from pydantic import Field

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    """Get the weather for a given location."""
    return f"The weather in {location} is sunny."

agent = Agent(
    client=client,
    instructions="You are a friendly assistant.",
    tools=[get_weather],
    default_options={"store": False},
)
```

- Annotate parameters with `Annotated[type, Field(description="...")]`
- The docstring becomes the tool description the model sees
- `approval_mode="never_require"` means the tool runs without user confirmation
- Pass all tools as a list to `Agent(tools=[...])`

### Add MCP Server Tools

Connect to remote tool servers using the Model Context Protocol:

```python
mcp_tool = client.get_mcp_tool(
    name="GitHub",
    url="https://api.githubcopilot.com/mcp/",
    headers={"Authorization": f"Bearer {token}"},
    approval_mode="never_require",
)

agent = Agent(client=client, instructions="...", tools=[mcp_tool], default_options={"store": False})
```

### Use Foundry Toolbox (Managed Tool Registry)

Load a shared, managed collection of tools from Foundry:

```python
import asyncio

async def main():
    toolbox = await client.get_toolbox(os.environ["TOOLBOX_NAME"])
    agent = Agent(client=client, instructions="...", tools=toolbox, default_options={"store": False})
    server = ResponsesHostServer(agent)
    await server.run_async()  # Use run_async() with asyncio

asyncio.run(main())
```

Note: When using async operations (toolbox loading), switch from `server.run()` to `await server.run_async()` and wrap in `asyncio.run()`.

### Build Multi-Agent Workflows

Chain multiple specialized agents in a pipeline:

```python
from agent_framework import Agent, AgentExecutor, WorkflowBuilder

writer = AgentExecutor(Agent(client=client, instructions="Write slogans.", name="writer"), context_mode="last_agent")
reviewer = AgentExecutor(Agent(client=client, instructions="Review for compliance.", name="reviewer"), context_mode="last_agent")
formatter = AgentExecutor(Agent(client=client, instructions="Format the output.", name="formatter"), context_mode="last_agent")

workflow_agent = (
    WorkflowBuilder(start_executor=writer, output_executors=[formatter])
    .add_edge(writer, reviewer)
    .add_edge(reviewer, formatter)
    .build()
    .as_agent()
)

ResponsesHostServer(workflow_agent).run()
```

- `context_mode="last_agent"` — each agent sees only the previous agent's output
- `output_executors` — defines which agent(s) produce the final response
- `.as_agent()` converts the workflow to a standard Agent, hosted the same way

### Declarative Workflows (YAML-Defined)

For complex routing logic, define workflows declaratively:

```python
from agent_framework_declarative import WorkflowFactory
from pydantic import BaseModel, Field
from typing import Literal

# Structured output model for routing decisions
class TriageResponse(BaseModel):
    Category: Literal["Technical", "Billing", "General"] = Field(description="...")
    Reply: str = Field(default="", description="...")

triage_agent = Agent(
    client=client, name="TriageAgent", instructions="...",
    default_options={"response_format": TriageResponse, "store": False},
)

factory = WorkflowFactory(agents={"TriageAgent": triage_agent, "SupportAgent": support_agent})
workflow = factory.create_workflow_from_yaml_path("workflow.yaml")
ResponsesHostServer(workflow.as_agent(name="my-workflow")).run()
```

The `workflow.yaml` uses kinds like `OnConversationStart`, `InvokeAzureAgent`, `ConditionGroup`, `SendActivity`, and `GotoAction` to define branching logic declaratively.

## Responses vs. Invocations Protocol

This sample uses the **Responses** protocol. The alternative is **Invocations**:

| Aspect | Responses (this sample) | Invocations |
|--------|------------------------|-------------|
| Endpoint | `POST /responses` | `POST /invocations` |
| History | Platform-managed (`previous_response_id`) | Self-managed (in-memory, Redis, etc.) |
| Streaming | Framework-managed SSE | Raw SSE — you format events |
| Best for | Most agents, OpenAI-compatible clients | Custom payloads, webhooks, GitHub Copilot Extensions |
| Server class | `ResponsesHostServer` | `InvocationAgentServerHost` |

Use Invocations when you need full control over the HTTP contract (custom JSON schemas, self-managed sessions, webhook integrations).

## Conventions

- **Python 3.12** with `python:3.12-slim` Docker base
- **Every sample follows the same file structure:** main.py, requirements.txt, agent.yaml, agent.manifest.yaml, Dockerfile, .env.example, README.md
- **Authentication:** Always use `DefaultAzureCredential()` — works with Azure CLI locally and managed identity in Foundry
- **History management:** Always set `store: False` in `default_options` for Responses protocol agents
- **Port 8088:** The ResponsesHostServer default; don't change it (Foundry expects it)
- **CODEOWNERS:** This path is owned by `@microsoft-foundry/hosted-agents`
- **Dependencies:** Keep requirements.txt minimal — only what's needed

## Broader Repository Context

This sample lives in `foundry-samples/samples/python/hosted-agents/agent-framework/responses/01-basic/`. The parent directories contain:

- `responses/` — Sibling samples 02-tools, 03-mcp, 04-foundry-toolbox, 05-workflows, 06-declarative-customer-support
- `invocations/` — Same Agent Framework but using the Invocations protocol
- `bring-your-own/` — Using other frameworks (LangGraph, CrewAI) with Foundry protocol SDKs
- Parent READMEs explain protocol choices, deployment workflows, and environment setup

## Foundry Deployment Model

When deployed to Foundry:
1. Docker image is built from the Dockerfile
2. Image is pushed to Azure Container Registry
3. Foundry runs the container with auto-injected environment variables
4. The agent is accessible via Foundry's managed endpoint
5. OpenTelemetry tracing is automatically configured via `APPLICATIONINSIGHTS_CONNECTION_STRING`
6. Health endpoints are provided by the hosting SDK

The `agent.manifest.yaml` declares what Foundry resources are needed (models), and `azd up` provisions everything.
