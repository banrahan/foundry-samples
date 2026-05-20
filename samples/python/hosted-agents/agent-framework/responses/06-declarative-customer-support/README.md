# Declarative Customer Support Workflow (Responses Protocol)

A realistic **multi-turn** [Agent Framework](https://github.com/microsoft/agent-framework) **declarative workflow** — defined entirely in YAML — hosted on Microsoft Foundry using the **Responses protocol**. It shows how a declarative workflow that invokes multiple Foundry-hosted agents can run end-to-end on every user turn while reading the prior conversation through `Conversation.messages` (populated automatically by `Workflow.as_agent()`).

## Prerequisites

1. **Azure Developer CLI (`azd`)** — [Install azd](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd)
2. Install the AI agent extension:
   ```bash
   azd ext install azure.ai.agents
   ```
3. Authenticate:
   ```bash
   azd auth login
   ```

## Quickstart

### Initialize the agent project

No cloning required. Create a new folder and initialize from the manifest:

```bash
mkdir my-support-agent && cd my-support-agent

azd ai agent init -m https://github.com/microsoft/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/06-declarative-customer-support/agent.manifest.yaml
```

Follow the prompts to configure your Foundry project and model deployment. If you don't have an existing Foundry project, `azd ai agent init` will guide you through creating one.

### Provision Azure resources (if needed)

If you don't already have a Foundry project and model deployment:

```bash
azd provision
```

### Run the agent locally

```bash
azd ai agent run
```

The agent host will start on `http://localhost:8088`.

### Invoke the local agent

In a separate terminal, from the project directory. A typical multi-turn session:

```bash
azd ai agent invoke --local "I have a problem"
# → "Could you tell me a bit more about what's going on?"

azd ai agent invoke --local "My laptop won't turn on"
# → "Connecting you with technical support..."
# → TechSupportAgent: "Let's start simple — is the charger LED on when plugged in?"

azd ai agent invoke --local "Yes the LED is on"
# → TechSupportAgent: "Great. Try a hard reset: hold the power button for 30 seconds..."
```

Or for billing:

```bash
azd ai agent invoke --local "I was double-charged this month"
# → "Connecting you with billing support..."
# → BillingAgent: "I'm sorry about that. Can you share the last 4 digits of the card on file?"
```

## Deploy to Foundry

Once tested locally, deploy to Microsoft Foundry:

```bash
azd deploy
```

For the full deployment guide, see [Deploy a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/deploy-hosted-agent).

### Invoke the deployed agent

```bash
azd ai agent invoke "I have a problem with my account"
```

## How it works

[`workflow.yaml`](workflow.yaml) describes a customer-support triage flow:

1. `InvokeAzureAgent: TriageAgent` — looks at the full conversation so far and emits a structured `TriageResponse` (`Category`, `NeedsClarification`, `ClarificationQuestion`, `Reply`).
2. `ConditionGroup` routes on the triage decision:
   - **NeedsClarification** → asks one focused follow-up question and ends the turn.
   - **Category = "Technical"** → hands off to `TechSupportAgent`.
   - **Category = "Billing"** → hands off to `BillingAgent`.
   - **else** → returns the triage agent's `Reply` directly (good for greetings or general questions).

[`main.py`](main.py) builds three `Agent` instances on top of a shared `FoundryChatClient` (one per workflow role), registers them with the `WorkflowFactory` so the YAML's `InvokeAzureAgent` actions can resolve them by name, loads the workflow, wraps it with `.as_agent(...)`, and hands the agent to `ResponsesHostServer`. See [main.py](main.py) and [workflow.yaml](workflow.yaml) for the implementation.

## Next steps

- [Quickstart: Create a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent) — end-to-end walkthrough using `azd`
- [Declarative workflows](https://learn.microsoft.com/en-us/agent-framework/workflows/declarative/?pivots=programming-language-python) — learn more about YAML-defined workflows
- [Workflow as an agent](https://learn.microsoft.com/en-us/agent-framework/workflows/as-agents?pivots=programming-language-python) — serving workflows via the Responses protocol
- [Manage hosted agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/manage-hosted-agent) — monitor and manage deployed agents
- [Basic agent](../01-basic/) — minimal agent with no tools
- [Programmatic workflows](../05-workflows/) — code-defined multi-agent pipeline
