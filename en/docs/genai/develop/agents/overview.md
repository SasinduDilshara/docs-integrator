---
sidebar_position: 1
title: AI Agents
description: Reference for AI Agents in WSO2 Integrator — the AI Chat Agent Wizard, system prompt, tools, memory, observability, and evaluations.
---

# AI Agents

An **AI Agent** is an integration component that combines an LLM, a system prompt, tools, and memory into a single artifact. Once configured, you call it with a user message and it reasons, calls tools, and produces a response. This section is the feature reference for everything an agent in WSO2 Integrator can be made of.

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [Creating an Agent](creating-an-agent.md) | The AI Chat Agent Wizard creates an agent service in one click — listener, agent node, return — and opens the canvas for configuration. | **Artifacts** → **AI Chat Agent**. |
| [Tools](tools.md) | The buttons the agent can press: existing functions, connectors, MCP servers, or custom tool definitions. | The agent canvas → **Add Tool** button on the AI Agent node, or the agent's right-side **Tools** panel. |
| [Memory](memory.md) | What the agent remembers across turns of a conversation. | **Add Memory** button on the AI Agent node → **Configure Memory**. |
| [Observability](observability.md) | Traces, logs, and metrics for every reasoning step and tool call. | **Tracing: Off/On** toggle at the top right of the agent canvas; standard Ballerina observability stack underneath. |
| [Evaluations](evaluations.md) | `bal test` plus LLM-as-judge — measure correctness, tool-use accuracy, and quality regressions in CI. | Standard `bal test` test files; pattern-based, no special UI required. |

## What an Agent Looks Like in the Canvas

Once created, an agent is a small flow with three blocks: **Start → AI Agent → Return**. The AI Agent block exposes everything you can configure about the agent in one place.

![The AI Agent canvas showing Start, an AI Agent node with the agent name and an Add Memory button, and a Return node. Right side has a Tracing toggle and a Chat button.](/img/genai/develop/agents/02-agent-flow-canvas.png)

The block surfaces:

- The agent's **role / name** (`BlogReviewer` in this example).
- A short **instructions placeholder** that opens the rich-text **Instructions** editor.
- An **Add Memory** button.
- An attached **Model Provider** (the small node on the right of the canvas, here `wso2ModelProvider`).
- The agent's **Tools** in a right-side panel (not visible in this frame).

The **Chat** button at the top right opens an in-IDE chat window so you can talk to the agent immediately, and the **Tracing** toggle controls whether each conversation produces a span tree you can inspect afterwards.

## When to Build an Agent

| Use an agent when… | Look elsewhere when… |
|---|---|
| The task takes multiple steps and the order isn't fixed in advance. | You know exactly which steps to run, in what order — write a normal flow. |
| The user is having a conversation, not making a single request. | One input, one answer — try a [natural function](/docs/genai/develop/natural-functions/overview) or a [direct LLM call](/docs/genai/develop/direct-llm/overview). |
| The agent should use existing connectors, functions, or MCP servers as tools. | The work has no lookup or action component — you're just shaping text. |
| You need to ground answers in your own documents *during* the conversation. | One-off retrieval — call a [RAG](/docs/genai/develop/rag/overview) flow directly. |

## Anatomy of the AI Agent Service

When you finish the **AI Chat Agent Wizard**, BI generates this in the project:

| Project artifact | What it does |
|---|---|
| `Listeners → chatAgentListener` | An `ai:Listener` that handles the chat protocol (request, session ID, response). |
| `Entry Points → AI Agent Services - /<name>` | The service your wizard created, exposed at `/<name>` over the listener. |
| `Connections → wso2ModelProvider` (or your chosen provider) | The model provider the agent uses. |
| Generated Ballerina source | A small `final ai:Agent ...` declaration plus a service that calls `agent.run(...)`. |

You can edit any of these later, but the wizard wires up the common shape so you don't have to.

### Listeners Reference

After the wizard runs, the project sidebar's **Listeners** section grows with the agent's listener. As you add more services (HTTP, MCP), each gets its own:

| Listener | Type | Created by | Used for |
|---|---|---|---|
| **chatAgentListener** | `ai:Listener` | AI Chat Agent Wizard | All agent services. Speaks the AI chat protocol — `ai:ChatReqMessage` in, `ai:ChatRespMessage` out. Handles per-session memory routing. |
| **httpDefaultListener** | `http:Listener` | First HTTP Service artifact | Plain HTTP services. Use this listener for RAG query endpoints, health checks, ingestion endpoints. |
| **mcpListener** | `mcp:Listener` | First MCP Service artifact | MCP services. Implements the Streamable HTTP variant of the Model Context Protocol. |

Click a listener in the sidebar to edit its configuration (port, host, base path, TLS, listener-level auth). Multiple services can share one listener — you don't need a new listener per artifact.

## Multi-Artifact Project View

A real project usually has more than one artifact: an agent for chat, plus an HTTP service for batch operations or RAG queries, plus an MCP service for assistants. The project Overview canvas visualizes all of them at once with the connections each one uses:

![Project Overview showing the Design canvas with two services side by side. Top row: chatAgentListener (ai:Listener) connected to AI Agent Service /blogReviewer connected to wso2ModelProvider (Model Provider). Bottom row: httpDefaultListener (http:Listener) connected to /api/v1 with [POST] /test resource, connected to aiWso2embeddingp... (Embedding Provider). Right side: Deployment Options (Deploy to WSO2 Cloud, Deploy with Docker, Deploy on a VM) and Integration Control Plane (Enable ICP monitoring, Publish to local ICP). Top-right buttons: Configure, Run, Debug.](/img/genai/develop/shared/14-multi-artifact-project-canvas.png)

Each artifact lives independently — they're not coupled by the canvas. The canvas just shows you who-connects-to-whom so you can spot orphan connections and shared providers at a glance.

The top-right toolbar has three actions:

| Action | What it does |
|---|---|
| **Configure** | Opens the project's `Config.toml` editor for editing configurables (API keys, endpoints, etc.). |
| **Run** | Compiles and runs the integration with `bal run --experimental` (the experimental flag is added automatically when natural functions are present). |
| **Debug** | Same as Run but attaches a debugger and shows breakpoint UI in the editor. |

## Generated Ballerina

For an agent named `blogReviewer` with the default WSO2 model provider, the generated source is roughly:

```ballerina
import ballerina/ai;

final ai:Agent blogReviewer = check new (
    systemPrompt = {
        role: "BlogReviewer",
        instructions: "..." // from the Instructions editor
    },
    tools = [...],         // your tools
    model = check ai:getDefaultModelProvider(),
    memory = ...           // your memory configuration
);

service /blogReviewer on chatAgentListener {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        return {message: check blogReviewer.run(request.message, request.sessionId)};
    }
}
```

You don't write this by hand, but the surface is intentionally small. If you switch to source view in BI, what you see is exactly what you'd expect.

## Try-It and Run Experience

The agent canvas's top-right buttons are how you exercise the agent without leaving BI:

| Button | What it does |
|---|---|
| **Tracing: Off / On** | Toggles OpenTelemetry tracing for this agent. When on, every conversation produces a span tree visible in the Tracing view. See [Observability](observability.md). |
| **Chat** | Opens an in-IDE chat surface — a small panel where you type a message, send it, and see the agent's response inline. The chat reuses a session ID across turns so memory works. Iterating on the system prompt while keeping the same conversation thread is the typical use. |

For project-level run controls (Configure / Run / Debug), see the [Multi-Artifact Project View](#multi-artifact-project-view) above. The Run button starts the integration; Debug starts it with the debugger attached so you can set breakpoints inside tools and natural-function bodies.

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent doesn't pick the tool you expected. | Tool description is vague, or the system prompt doesn't mention when to use the tool. | Tighten the tool description; add a one-liner trigger condition in Instructions. |
| Agent's first response is empty, second is fine. | The default WSO2 Model Provider isn't fully signed in yet. | Run **Ballerina: Configure default WSO2 model provider** from the Command Palette. |
| Agent drifts off-topic over a long conversation. | Memory is full and trimming is dropping the system prompt context. | Lower **Max Messages Per Key** (MSSQL) or use a larger-context model. |
| Same input produces wildly different responses. | Temperature is high on the model provider. | Lower temperature on the provider's Advanced Configurations. |
| `bal run` fails with "default model provider not configured". | `wso2aiKey` missing from `Config.toml`. | Run **Configure default WSO2 model provider** again. |

## What's Next

- **[Creating an Agent](creating-an-agent.md)** — the AI Chat Agent Wizard, role, and instructions.
- **[Tools](tools.md)** — give the agent buttons it can press.
- **[Memory](memory.md)** — multi-turn conversations.
- **[Observability](observability.md)** — see what your agent is doing in production.
- **[Evaluations](evaluations.md)** — keep agent quality from regressing.
- **[What is an AI Agent?](/docs/genai/key-concepts/what-is-ai-agent)** — conceptual background.
