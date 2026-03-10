---
sidebar_position: 1
title: "Agent Architecture & Concepts"
description: Understand how agents work in WSO2 Integrator — the agent loop, memory types, tool calling, and the difference between chat agents and inline agents.
---

# Agent Architecture & Concepts

Agents in WSO2 Integrator are autonomous components that perceive user input, reason about what to do, and act by calling tools or generating responses. You build agents using the same visual designer and pro-code experience as any other integration artifact, but they are backed by LLMs for decision-making.

This page covers the foundational concepts you need before building your first agent. If you want to jump straight into code, start with the [Build a Conversational Agent](/docs/genai/quick-starts/conversational-agent) quick start.

## The Agent Loop

Every agent in WSO2 Integrator follows a **perceive-reason-act** loop:

1. **Perceive** — The agent receives input (a user message, an API request, or an event).
2. **Reason** — The LLM processes the input along with conversation history, system instructions, and available tool descriptions. It decides whether to respond directly or call one or more tools.
3. **Act** — The agent either returns a response to the user or executes tool calls. If tools were called, their results feed back into the reasoning step for another iteration.

```
┌─────────────────────────────────────────┐
│              Agent Loop                 │
│                                         │
│   User Input ──► Perceive               │
│                     │                   │
│                  Reason (LLM)           │
│                   ╱     ╲               │
│            Respond     Call Tools       │
│               │           │             │
│            Output    Tool Results ──┐   │
│                                     │   │
│                  Reason (LLM) ◄─────┘   │
│                     │                   │
│                  Respond                │
│                     │                   │
│                  Output                 │
└─────────────────────────────────────────┘
```

The loop continues until the LLM decides it has enough information to respond, or until a maximum iteration limit is reached.

:::info
The agent loop is not a simple request-response cycle. A single user message can trigger multiple rounds of reasoning and tool calling before a final answer is produced.
:::

## Agents vs. Direct LLM Calls

It is important to understand when to use an agent versus a simple LLM call:

| Capability | Direct LLM Call | Agent |
|---|---|---|
| Single question/answer | Yes | Overkill |
| Multi-turn conversation | Manual | Built-in |
| Tool calling | Manual orchestration | Automatic |
| Memory management | Manual | Configurable |
| Streaming responses | Yes | Yes |
| Autonomous decision-making | No | Yes |

Use a **direct LLM call** (or a [natural function](/docs/genai/agents/natural-functions)) when you need a one-shot transformation or classification. Use an **agent** when the task requires reasoning, tool use, or multi-turn interaction.

## Agent Types

WSO2 Integrator supports two primary agent types:

### Chat Agents

Chat agents are user-facing agents exposed through a REST API with built-in chat semantics. They are ideal for conversational interfaces — chatbots, support agents, personal assistants.

```ballerina
final agent:ChatAgent mathTutor = check new (
    systemPrompt = "You are a patient math tutor. Explain step by step.",
    model = check new openai:Client({auth: {token: apiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 50),
    tools = [solveEquation, plotGraph]
);
```

Key characteristics:
- Automatic REST endpoint with `/chat` resource
- Built-in chat interface for testing
- Conversation memory handled automatically
- Streaming response support

See [Create a Chat Agent](/docs/genai/agents/chat-agents) for a full walkthrough.

### Inline Agents (API-Exposed)

Inline agents are embedded directly into your service logic — inside HTTP handlers, GraphQL resolvers, or automation flows. They do not expose their own endpoint; instead, you call them programmatically.

```ballerina
service /graphql on new graphql:Listener(4000) {

    resource function get recommendation(string userQuery) returns string|error {
        agent:InlineAgent recommender = check new (
            systemPrompt = "You recommend products based on user preferences.",
            model = check new openai:Client({auth: {token: apiKey}}),
            tools = [searchProducts, getReviews]
        );
        agent:ChatMessage response = check recommender.run(userQuery);
        return response.content;
    }
}
```

Key characteristics:
- No dedicated endpoint — lives inside existing service logic
- Fine-grained control over when and how the agent runs
- Suitable for GraphQL resolvers, webhook handlers, and automation steps

See [Expose an Agent as an API](/docs/genai/agents/api-exposed-agents) for details.

## Memory Types

Memory determines what context the agent has access to during reasoning. WSO2 Integrator provides three memory strategies:

### Conversation Memory

Stores the most recent N messages in the current session. Simple and effective for short interactions.

```ballerina
memory = new agent:MessageWindowChatMemory(maxMessages = 20)
```

### Semantic Memory

Uses a vector store to retrieve relevant past interactions based on similarity. Best for agents that need to recall specific topics from long conversation histories.

```ballerina
memory = new agent:SemanticChatMemory(vectorStore = myVectorStore, topK = 5)
```

### Persistent Memory

Stores conversation history in a database for cross-session recall. Use this when users return to continue previous conversations.

```ballerina
memory = new agent:PersistentChatMemory(store = myDatabaseStore)
```

See [Configure Agent Memory](/docs/genai/agents/memory-configuration) for detailed configuration options.

## Tool Calling

Tools give agents the ability to interact with the outside world. A tool is a typed function that the LLM can choose to invoke based on the user's request and the tool's description.

```ballerina
@agent:Tool
isolated function searchDatabase(string query, int maxResults = 10) returns record{}[]|error {
    // The agent decides when and how to call this
    return db->query(query, maxResults);
}
```

Tools can be generated from:
- **Existing Ballerina functions** — Annotate with `@agent:Tool`
- **Connector actions** — Expose connector methods as agent tools
- **OpenAPI specifications** — Auto-generate tools from API specs

See [Bind Tools to Agents](/docs/genai/agents/tool-binding) for the complete guide.

## What's Next

- [Create a Chat Agent](/docs/genai/agents/chat-agents) — Build your first user-facing conversational agent
- [Expose an Agent as an API](/docs/genai/agents/api-exposed-agents) — Embed agents inside HTTP and GraphQL services
- [Build a Conversational Agent](/docs/genai/quick-starts/conversational-agent) — Hands-on quick start with a working example
