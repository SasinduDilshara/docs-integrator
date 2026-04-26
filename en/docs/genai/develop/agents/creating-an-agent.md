---
sidebar_position: 2
title: Creating an Agent
description: Reference for the AI Chat Agent Wizard in WSO2 Integrator — the agent service, listener, role, and instructions.
---

# Creating an Agent

WSO2 Integrator creates an AI agent in one shot through the **AI Chat Agent Wizard**. The wizard generates a service, a listener, an agent node, and the surrounding flow — and drops you onto the canvas where you configure the agent's role, instructions, model, tools, and memory.

This page covers everything that happens inside the wizard and the canvas. For the parts on the right side of the canvas — Tools, Memory, and so on — see the dedicated pages.

## Launching the Wizard

1. Open your integration project in BI.
2. Click **+ Add Artifact** at the top right of the project view, or right-click the project tree.
3. The **Artifacts** page opens. Pick **AI Chat Agent**.

![Artifacts page listing all artifact types — HTTP Service, GraphQL Service, TCP Service, Event Integration, File Integration, Function, Data Mapper, Type, Connection, Configuration. AI Chat Agent appears under Integration as API.](/img/genai/develop/shared/03-add-artifact-page.png)

The wizard opens with a single field.

## The Wizard Form

| Field | Required | Notes |
|---|---|---|
| **Name** | Yes | Plain identifier for the agent: `BlogReviewer`, `SupportAssistant`, `SalesAdvisor`. Used for the service path (`/blogReviewer`), the agent variable, and the breadcrumb. The placeholder text suggests example names. |

That's the whole form. Click the **Create** button and BI does several things at once:

![The Create AI Chat Agent wizard with a Name field set to BlogReviewer and a Creating… spinner showing 'Configuring the service listener'.](/img/genai/develop/agents/01-create-ai-chat-agent-wizard.png)

What it generates:

- A **listener** (`chatAgentListener`) configured for the AI chat protocol.
- A **service** at `/<name>` (e.g. `/blogReviewer`).
- A **resource function** that takes a chat request, calls the agent, and returns the response.
- An **agent declaration** with placeholders for instructions, tools, and memory.
- A **default model provider** if your project doesn't already have one.

After creation, BI navigates straight to the agent canvas.

## The Agent Canvas

The canvas is a tiny three-block flow: **Start → AI Agent → Return**.

![The AI Chat Agent canvas with Start, an AI Agent node showing the agent name BlogReviewer and the placeholder 'Provide specific instructions on how the agent should behave.' plus an Add Memory button, and a Return node. The agent is connected to a wso2ModelProvider on the right. Top-right buttons: Tracing: Off, Chat.](/img/genai/develop/agents/02-agent-flow-canvas.png)

The AI Agent node exposes everything you typically configure:

| Element | What it controls |
|---|---|
| **Role / Name** | Shown at the top of the node (`BlogReviewer`). Doubles as the agent's role in the system prompt. |
| **Instructions placeholder** | Click to open the rich-text **Instructions** editor (the system prompt). |
| **+ Add Memory** | Configure conversation memory ([details](memory.md)). |
| **Connection line** to a Model Provider | The LLM the agent uses. The wizard wires this up automatically; you can change it from the right-side panel. |
| **Tools** (right-side panel) | The functions, connectors, MCP servers, and custom tools the agent can call ([details](tools.md)). |

At the top right of the canvas:

- **Tracing: Off / On** — toggle OpenTelemetry tracing for the agent ([observability](observability.md)).
- **Chat** — open an in-IDE chat window to talk to the agent immediately. Useful for iterating on instructions and tools.

## Writing the Instructions (System Prompt)

Click the Instructions placeholder on the AI Agent node to open the editor:

![The Instructions editor — a rich-text panel with Insert, Bold, Italic, Link, H1, Quote, list, table, and an AI assist icon. Placeholder reads 'e.g., You are a friendly assistant. Your goal is to...'. Preview / Source toggle on the top right.](/img/genai/develop/agents/03-instructions-editor.png)

The Instructions editor is a Markdown-aware rich-text surface. The text is stored as the agent's system prompt and sent on every turn.

A practical Instructions skeleton:

```
You are <role> for <organisation>.

Goals
- <what success looks like>
- <secondary goal>

Rules
- <hard rule 1>
- <hard rule 2>
- Never <thing the agent must not do>

Style
- Tone: <warm / formal / concise>
- Length: <one paragraph / under 150 words>

Tool usage
- Use <toolName> when <condition>.
- Confirm with the user before calling <destructiveTool>.
```

Tips:

- **Goals first, then rules.** Models follow ordered constraints better than tangled paragraphs.
- **Be explicit about scope.** *"Only answer questions about orders, returns, and products."* prevents off-topic answers more reliably than hoping the model will figure it out.
- **Mention each tool the agent has, with a one-liner trigger condition.** It dramatically improves tool-selection accuracy.
- **Don't restate the output schema** when the response is plain text — let the model write naturally.

The **Preview / Source** toggle lets you switch between the Markdown rendering and the raw template source if you want to paste a prompt in from elsewhere.

## Configuring the Model

Every agent has a single model provider on the **Model** field. The wizard wires up the default WSO2 provider; to change it:

1. On the right-hand canvas panel open **Model Providers** (or click the model node at the right of the AI Agent block).
2. Pick a different connection, or add a new one (see [Adding a Model Provider](/docs/genai/develop/direct-llm/adding-a-model-provider)).

Sampling parameters (temperature, top-p, max tokens) live on the model provider, not on the agent. To make the agent more deterministic, lower the temperature on its provider.

## Iteration Limit (`maxIter`)

An agent uses a Reason → Act → Observe loop. The **maxIter** setting caps how many loops it will run before giving up and answering with whatever it has.

| Default | Typical values |
|---|---|
| 5 | 3–5 for chat agents (snappy, simple answers). 10–15 for research-style agents that may need many tool calls. |

You can change `maxIter` from the agent's source view, or it can be added as a configurable on the agent — talk to your project lead about which fits the codebase.

## What's Next

- **[Tools](tools.md)** — give the agent something to do.
- **[Memory](memory.md)** — keep context across turns.
- **[Observability](observability.md)** — see and debug what the agent does.
- **[Evaluations](evaluations.md)** — protect quality with tests.
