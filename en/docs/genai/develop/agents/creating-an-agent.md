---
sidebar_position: 2
title: Creating an Agent
description: Reference for the AI Chat Agent Wizard in WSO2 Integrator — the agent service, listener, role, instructions, query binding, and result variable.
---

# Creating an Agent

WSO2 Integrator creates an AI agent in one shot through the **AI Chat Agent Wizard**. The wizard generates a service, a listener, an agent node, and the surrounding flow — and drops you onto the canvas where you configure the agent's role, instructions, model, query binding, tools, and memory.

This page covers everything that happens inside the wizard, the canvas, and the AI Agent node configuration panel. For Tools, Memory, and Observability — see the dedicated pages.

## Launching the Wizard

1. Open your integration project in BI.
2. Click **+ Add Artifact** at the top right of the project view, or right-click the project tree.
3. The **Artifacts** page opens.

![Artifacts page in BI showing all artifact categories — Automation, AI Integration (AI Chat Agent, MCP Service), Integration as API (HTTP Service, GraphQL Service Beta, TCP Service Beta), Event Integration (Kafka, RabbitMQ, MQTT, Azure Service Bus, Salesforce, Twilio, GitHub, Solace, CDC for Microsoft SQL Server, CDC for PostgreSQL).](/img/genai/develop/shared/07-artifacts-page-full.png)

4. Under **AI Integration**, pick **AI Chat Agent**.

The Artifacts page groups every kind of integration WSO2 Integrator supports. The AI Integration category contains exactly two artifacts: **AI Chat Agent** and **MCP Service**. Other AI capabilities — direct LLM calls, natural functions, and RAG — are not artifact types of their own; they're nodes you add inside any flow.

Scroll further on the same page to see the rest of the artifact catalogue:

![Artifacts page scrolled to File Integration (FTP/SFTP, Local Files) and Other Artifacts (Function, Data Mapper, Type, Connection, Configuration).](/img/genai/develop/shared/13-artifacts-other-categories.png)

## The Wizard Form

The wizard opens with a single field, but the **Create** button stays disabled until you provide a name.

![The empty AI Chat Agent wizard with a Name field and a disabled grey Create button. The placeholder shows example names: 'Customer Support Assistant', 'Sales Advisor', 'Data Analyst'.](/img/genai/develop/agents/01-create-ai-chat-agent-wizard.png)

| Field | Required | Notes |
|---|---|---|
| **Name** | Yes | Plain identifier for the agent: `BlogReviewer`, `SupportAssistant`, `SalesAdvisor`. Used for the service path (`/blogReviewer`), the agent variable, and the breadcrumb. The placeholder text suggests example names. Must start with a letter and contain only letters, digits, and underscores. |

Type a name (e.g. `BlogReviewer`) and the **Create** button becomes active. Click it; BI shows a "Configuring the service listener" spinner while it generates the artifacts.

What it generates in one click:

- A **listener** (`chatAgentListener`) configured for the AI chat protocol.
- A **service** at `/<name>` (e.g. `/blogReviewer`).
- A **resource function post chat** that takes an `ai:ChatReqMessage`, calls the agent, and returns an `ai:ChatRespMessage`.
- An **agent declaration** with placeholders for instructions, tools, and memory.
- A **default model provider** connection if your project doesn't already have one.

After creation, BI navigates straight to the agent canvas.

## The Agent Canvas

The canvas is a tiny three-block flow: **Start → AI Agent → Return**.

![The AI Chat Agent canvas with Start, an AI Agent node showing the agent name BlogReviewer and the placeholder 'Provide specific instructions on how the agent should behave.' plus an Add Memory button, and a Return node. The agent is connected to a wso2ModelProvider on the right. Top-right buttons: Tracing: Off, Chat.](/img/genai/develop/agents/02-agent-flow-canvas.png)

The AI Agent node exposes everything you typically configure:

| Element | What it controls |
|---|---|
| **Role / Name** | Shown at the top of the node (`BlogReviewer`). Doubles as the agent's role in the system prompt. |
| **Instructions placeholder** | Click anywhere on the placeholder text or the node body to open the right-side AI Agent configuration panel — see [The AI Agent Node Configuration Panel](#the-ai-agent-node-configuration-panel) below. |
| **+ Add Memory** | Configure conversation memory ([details](memory.md)). |
| **Connection line** to a Model Provider | The LLM the agent uses. The wizard wires this up automatically; click it to change. |
| **Tools** (right-side panel) | The functions, connectors, MCP servers, and custom tools the agent can call ([details](tools.md)). |
| **`+`** button on the right edge of the AI Agent block | Quick-add a tool or memory without opening the right panel. |
| **`⋮` menu** on the AI Agent block | Options for the node — edit, delete, view source. |

At the top right of the canvas:

- **`↶ ↷`** — undo / redo for the flow editor.
- **Tracing: Off / On** — toggle OpenTelemetry tracing for the agent ([observability](observability.md)).
- **Chat** — open an in-IDE chat window to talk to the agent immediately. Useful for iterating on instructions and tools.

At the bottom-left corner:

- **`⛶`** — fit-to-screen.
- **`+ / -`** — zoom in / out.

## The AI Agent Node Configuration Panel

Click the AI Agent node body and the configuration panel slides in on the right. This is where every per-call setting lives:

![The right-side AI Agent configuration panel. Top: 'AI Agent — Executes the agent for a given user query.' Fields visible: Role (with Prompt/Expression toggle), Instructions (with Prompt/Expression toggle, pre-filled with '# Role You are a blog content reviewer specializing in technical and editorial quality. # Task Review the blog below and provide a structured assessment.'), Query* (with Text/Expression toggle, default value `request.message`), Advanced Configurations Expand link, Result* (default value 'stringResult'), Save button.](/img/genai/develop/agents/11-ai-agent-config-panel.png)

| Field | Required | What it does |
|---|---|---|
| **Role** | Yes | The agent's role string. Pre-filled from the wizard name (e.g. `BlogReviewer`). Shown at the top of the agent box and used as the role in the generated system prompt. **Prompt / Expression** toggle lets you bind it to an expression instead. |
| **Instructions** | Yes | The full system prompt. Supports Markdown, multi-line text, and `${...}` interpolation against in-scope values. **Prompt / Expression** toggle lets you bind it to a string expression — useful when instructions come from a configurable. See [Writing Instructions](#writing-instructions) below. |
| **Query** | Yes | The user query expression — what the agent receives on each call. Defaults to `request.message` (the message field of the incoming `ai:ChatReqMessage`). **Text / Expression** toggle lets you wrap or transform the query before it reaches the agent. |
| **Advanced Configurations** | No | Per-call overrides. Expand to see additional options. |
| **Result** | Yes | The variable name where the agent's response is stored. Defaults to `stringResult`. |

Click **Save** to apply the configuration. Saving regenerates the underlying Ballerina code immediately.

## Writing Instructions

Click the **Instructions** field to open the rich-text editor on top of the canvas:

![The Instructions editor opened over the canvas. Rich-text toolbar with Insert dropdown, Bold/Italic/Link, H1, Quote, bullet list, numbered list, table, AI assist (magic wand) icon, and Preview/Source toggle. Placeholder text reads 'e.g., You are a friendly assistant. Your goal is to...'.](/img/genai/develop/agents/03-instructions-editor.png)

| Tool | What it does |
|---|---|
| **Insert** | Drop in references to data already in scope — see [The Insert Menu](#the-insert-menu) below. |
| **B / I / Link** | Bold, italic, hyperlink. Rendered as text in the saved prompt. |
| **H1 / dropdown** | Heading levels (H1, H2, H3, …). Useful for structuring long prompts. |
| **""** (quote) | Block quote. |
| **Bullet / numbered list** | Standard lists. |
| **Table** | Markdown table editor. Useful for "if user mentions X, do Y" tables. |
| **Magic wand** | AI prompt assist — describe the agent's job in one line and BI suggests a system-prompt scaffold. |
| **Preview / Source** | Switch between the rendered preview and the raw Markdown / template source. Use **Source** when you want to paste a long prompt in from somewhere else. |
| **`⛶`** (top-right of the editor) | Expand the editor to fill the window. |

### The Insert Menu

Click **Insert** to drop a reference into the prompt without typing it:

![The Insert dropdown open inside the Instructions editor showing four sub-categories: Inputs, Variables, Configurables, Functions — each with a chevron indicating a sub-menu.](/img/genai/develop/agents/12-instructions-insert-menu.png)

| Sub-menu | What you can insert |
|---|---|
| **Inputs** | Parameters of the resource that calls the agent — for example `request.sessionId`, `request.message`. |
| **Variables** | In-scope local variables. |
| **Configurables** | `configurable` values from the project's `Config.toml`. Useful for environment-specific instructions (e.g. a `tenantId` baked into the system prompt). |
| **Functions** | In-scope functions — call them inside `${...}` interpolations. |

Each picked entry is inserted into the editor as a `${...}` interpolation that evaluates at call time.

### A Practical Instructions Skeleton

```
# Role
You are a <role> for <organisation>.

# Goals
- <what success looks like>
- <secondary goal>

# Rules
- <hard rule 1>
- <hard rule 2>
- Never <thing the agent must not do>

# Style
- Tone: <warm / formal / concise>
- Length: <one paragraph / under 150 words>

# Tool usage
- Use ${toolName} when <condition>.
- Confirm with the user before calling ${destructiveTool}.
```

Tips:

- **Goals first, then rules.** Models follow ordered constraints better than tangled paragraphs.
- **Be explicit about scope.** *"Only answer questions about orders, returns, and products."* prevents off-topic answers more reliably than hoping the model will figure it out.
- **Mention each tool the agent has, with a one-liner trigger condition.** It dramatically improves tool-selection accuracy.
- **Don't restate the output schema** when the response is plain text — let the model write naturally.

## Configuring the Model

Every agent has exactly one model provider on the **Model** field. The wizard wires the default WSO2 provider automatically; to change it click the model node on the right of the AI Agent block.

The **Configure Model Provider** panel opens. This is **not** the same as creating a model provider — it picks an existing one or links to the creation flow:

![The Configure Model Provider panel showing 'Select Model Provider*' dropdown with wso2ModelProvider, a '+ Create New Model Provider' link below, and an info banner: 'Configure this model provider using the VS Code command palette command: > Ballerina: Configure default WSO2 model provider'. Save button at the bottom.](/img/genai/develop/agents/13-configure-model-provider.png)

| Field | What it does |
|---|---|
| **Select Model Provider*** | Pick from existing model-provider connections in the project. **Select / Expression** toggle lets you bind to an expression. |
| **+ Create New Model Provider** | Link to add a brand-new provider connection without leaving the panel — opens the Select Model Provider picker. See [Components → Model Providers](/docs/genai/develop/components/model-providers) for the full per-provider form reference. |
| **Save** | Wires the chosen provider into the agent. |

Sampling parameters (temperature, top-p, max tokens) live on the **model provider connection**, not on the agent. To make the agent more deterministic, lower the temperature on its provider via Connections → edit the provider → Advanced Configurations.

## Iteration Limit (`maxIter`)

An agent uses a Reason → Act → Observe loop. The **maxIter** setting caps how many loops it will run before giving up and answering with whatever it has.

| Default | Typical values |
|---|---|
| 5 | 3–5 for chat agents (snappy, simple answers). 10–15 for research-style agents that may need many tool calls. |

`maxIter` is exposed in the AI Agent node's **Advanced Configurations** section. You can also override it from source view, or expose it as a `configurable` so each environment uses a different ceiling.

## After-State: Agent with Memory and Tools Wired Up

Once you've added a memory and at least one tool, the AI Agent block grows extra rows inside it — making the configuration visible at a glance:

![The AI Chat Agent canvas after Memory has been attached. The AI Agent block now has an additional inner block 'Memory: ShortTermMemory' between the agent name and the BlogReviewer label.](/img/genai/develop/agents/22-agent-with-memory-attached.png)

The block now shows:

- **AI Agent / `stringResult`** at the top.
- A **Memory: ShortTermMemory** sub-block.
- The agent's role label (`BlogReviewer`).
- The Instructions placeholder.

The same pattern applies for tools, MCP toolkits, and any custom toolkit you add — each appears as its own row inside the agent block. Click any row to edit it; click the `⋮` to remove it.

## Project View Integration Diagram

Once an agent service exists, the project's **Overview** page shows it visually as part of the integration diagram:

![Project Overview showing the Design canvas with chatAgentListener (ai:Listener) connected to AI Agent Service /blogReviewer, which is connected to wso2ModelProvider (Model Provider). The right side shows Deployment Options (Deploy to WSO2 Cloud, Deploy with Docker, Deploy on a VM) and Integration Control Plane (Enable ICP monitoring, Publish to local ICP). Top-right buttons: Configure, Run, Debug. + Add Artifact button. Generate with AI button.](/img/genai/develop/shared/12-project-design-canvas.png)

The integration design canvas shows:

- **Listeners** on the left (`chatAgentListener`).
- **Entry Points** in the middle (the AI Agent Service).
- **Connections** on the right (the model provider, vector knowledge base, etc.).
- **Lines** showing which entry points use which connections.

This view is read-only at the project level — clicking a node opens the relevant artifact for editing. Deployment Options and Integration Control Plane on the right side are for shipping the integration to WSO2 Cloud, Docker, a VM, or the WSO2 Integrator Console.

## Talking to the Agent

The **Chat** button at the top right of the agent canvas opens an in-IDE chat surface so you can iterate on the agent without writing a separate test client. The chat panel:

- Sends each message to the running agent service through the listener.
- Generates and reuses a `sessionId` so memory works.
- Shows the agent's responses as a regular chat thread.
- Survives across canvas-edit cycles, so you can tweak Instructions or Tools and continue the same conversation.

For non-interactive testing — assertions on tool selection, snapshot comparisons, LLM-as-judge scoring — see [Evaluations](evaluations.md).

## What's Next

- **[Tools](tools.md)** — give the agent something to do.
- **[Memory](memory.md)** — keep context across turns.
- **[Observability](observability.md)** — see and debug what the agent does.
- **[Evaluations](evaluations.md)** — protect quality with tests.
