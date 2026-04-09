---
sidebar_position: 3
title: "Adding Memory to an Agent"
description: Set up conversation, semantic, and persistent memory for agents to manage context across interactions.
---

# Adding Memory to an Agent

Memory determines what context an agent has access to when reasoning about a user's message. The right memory configuration can make the difference between an agent that forgets everything after five messages and one that recalls relevant details from weeks ago.

WSO2 Integrator provides three memory types, each suited to different scenarios. You can also combine them for agents that need both short-term recall and long-term knowledge.

## Conversation Memory (Message Window)

Conversation memory keeps the most recent N messages in the current session. It is the simplest and most commonly used memory type.

```ballerina
import ballerinax/ai.agent;

final agent:ChatAgent assistant = check new (
    systemPrompt = "You are a helpful assistant.",
    model = myModel,
    memory = new agent:MessageWindowChatMemory(maxMessages = 20)
);
```

### Configuration Options

| Option | Type | Default | Description |
|---|---|---|---|
| `maxMessages` | `int` | `20` | Maximum number of messages to retain |
| `includeSystemPrompt` | `boolean` | `true` | Whether the system prompt counts toward the window |

### How It Works

1. Each new user message and agent response is appended to the history.
2. When the history exceeds `maxMessages`, the oldest messages are removed.
3. The system prompt is always included unless explicitly excluded.

### When to Use

- Short, focused conversations (support tickets, quick Q&A)
- Agents where older context is not important
- Low token usage is a priority

:::warning
Messages dropped from the window are gone permanently within that session. If an agent needs to reference earlier context, consider semantic memory instead.
:::

## Semantic Memory

Semantic memory stores conversation history in a vector store and retrieves relevant past messages based on similarity to the current query. This lets the agent recall specific topics from long conversations without keeping the entire history in context.

```ballerina
import ballerinax/ai.agent;
import ballerinax/ai.rag;

final rag:InMemoryVectorStore memoryStore = new;

final agent:ChatAgent assistant = check new (
    systemPrompt = "You are a research assistant with long-term recall.",
    model = myModel,
    memory = new agent:SemanticChatMemory(
        vectorStore = memoryStore,
        embeddingModel = myEmbeddingModel,
        topK = 5,
        recentWindowSize = 5
    )
);
```

### Configuration Options

| Option | Type | Default | Description |
|---|---|---|---|
| `vectorStore` | `VectorStore` | Required | The vector store for embedding and retrieving messages |
| `embeddingModel` | `EmbeddingModel` | Required | Model used to embed messages |
| `topK` | `int` | `5` | Number of relevant past messages to retrieve |
| `recentWindowSize` | `int` | `5` | Number of most recent messages always included |

### How It Works

1. Every message is embedded and stored in the vector store.
2. For each new user message, the agent retrieves the `topK` most semantically similar past messages.
3. The `recentWindowSize` most recent messages are always included, regardless of similarity.
4. Retrieved messages and recent messages are merged and passed to the LLM.

### When to Use

- Long-running conversations where specific topics may resurface
- Research or advisory agents where users revisit earlier questions
- Agents that handle multiple topics in a single session

:::tip
Combine a small `recentWindowSize` (3-5 messages) with a larger `topK` (5-10) to keep immediate context tight while still recalling relevant history.
:::

## Persistent Memory

Persistent memory stores conversation history in a database, enabling agents to recall context across sessions. When a user returns days later, the agent can pick up where they left off.

```ballerina
import ballerinax/ai.agent;
import ballerinax/postgresql;

configurable string dbUrl = ?;

final postgresql:Client dbClient = check new (dbUrl);

final agent:ChatAgent assistant = check new (
    systemPrompt = "You are a personal assistant that remembers user preferences.",
    model = myModel,
    memory = new agent:PersistentChatMemory(
        store = new agent:DatabaseChatStore(dbClient),
        sessionIdExtractor = "userId",
        maxMessages = 50
    )
);
```

### Configuration Options

| Option | Type | Default | Description |
|---|---|---|---|
| `store` | `ChatStore` | Required | Database-backed storage implementation |
| `sessionIdExtractor` | `string` | Required | Field used to identify the user/session |
| `maxMessages` | `int` | `50` | Maximum messages to load per session |
| `ttl` | `decimal` | `0` (no expiry) | Time-to-live for stored messages in seconds |

### How It Works

1. All messages are persisted to the configured database.
2. When a session resumes, the agent loads the most recent `maxMessages` from storage.
3. The `sessionIdExtractor` maps incoming requests to stored sessions.
4. Optional TTL automatically cleans up old conversations.

### When to Use

- Multi-session interactions (users return over days/weeks)
- Agents that need to remember user preferences or history
- Compliance scenarios requiring conversation audit trails

## Combining Memory Types

For advanced scenarios, you can layer memory types. For example, use conversation memory for the current session and persistent memory for cross-session recall:

```ballerina
final agent:ChatAgent assistant = check new (
    systemPrompt = "You are a personal advisor.",
    model = myModel,
    memory = new agent:CompositeMemory(
        shortTerm = new agent:MessageWindowChatMemory(maxMessages = 10),
        longTerm = new agent:PersistentChatMemory(
            store = new agent:DatabaseChatStore(dbClient),
            sessionIdExtractor = "userId"
        )
    )
);
```

## Memory Selection Guide

| Scenario | Recommended Memory | Reasoning |
|---|---|---|
| Quick support chat | Conversation (10-20 messages) | Short interactions, no recall needed |
| Research assistant | Semantic (topK=10) | Users revisit topics within a session |
| Personal advisor | Persistent + Conversation | Cross-session recall with recent context |
| One-shot task agent | None (inline agent) | No memory needed |
| High-compliance domain | Persistent (with TTL) | Audit trail with automatic cleanup |

## Configuring Memory via the Visual Designer

1. By default, agents come preconfigured with an in-memory implementation.
2. To change the memory type, click **Add Memory** in the agent configuration panel and select your desired option.

:::info
The default in-memory implementation stores conversation history for the duration of the running session. When the service restarts, history is cleared. For cross-session recall, use persistent memory.
:::

## What's Next

- [Adding Tools to an Agent](adding-tools-to-an-agent.md) -- Give agents access to external services and data
- [Agent Configuration Options](agent-configuration-options.md) -- Multi-agent orchestration and advanced configuration
- [Creating an AI Agent](creating-an-ai-agent.md) -- Review the fundamentals of agent creation
