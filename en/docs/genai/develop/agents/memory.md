---
sidebar_position: 4
title: Memory
description: Reference for configuring AI agent memory in WSO2 Integrator — short-term in-memory, MS SQL, custom stores, and overflow strategy.
---

# Memory

**Memory** is what lets your agent remember earlier turns of a conversation. WSO2 Integrator handles per-session memory automatically when an agent runs on `ai:Listener` — but the **Configure Memory** dialog lets you tune *what* is remembered, *where* it's stored, and *how long* it survives.

## The Configure Memory Panel

Click **+ Add Memory** on the AI Agent node. The **Configure Memory** panel opens on the right:

![The Configure Memory panel — Select Memory dropdown set to Short Term Memory, with description 'Initializes short-term memory with an optional store and overflow configuration.' Advanced Configurations section with Store (Default: In-Memory Short Term Memory Store), Overflow Configuration (Default: Overflow Trim), Memory Name set to aiShorttermmemory. Save button at the bottom.](/img/genai/develop/agents/09-configure-memory.png)

| Field | Required | What it does |
|---|---|---|
| **Select Memory** | Yes | Memory **strategy** — currently **Short Term Memory** is the standard choice. |
| **Store** | No | Where the messages are persisted. Defaults to In-Memory; other options listed below. |
| **Overflow Configuration** | No | What happens when the memory window is full. Defaults to **Overflow Trim** (drop oldest). |
| **Memory Name** | Yes | The variable name in the generated source. Defaults to `ai<Type>memory`. |

After saving, the AI Agent node shows the memory connection in its node body and the right-side connections panel.

## Memory Stores

The **Store** field decides where the conversation history actually lives. Click the field to open the **Select Memory Store** picker:

![The Select Memory Store panel listing two stores: In Memory Short Term Memory Store with description 'Provides an in-memory chat message store.' (highlighted) and MSSQL Short Term Memory Store with description 'Represents an MS SQL-backed short-term memory store for messages.'](/img/genai/develop/agents/10-select-memory-store.png)

| Store | Module | Survives a restart? | Good for |
|---|---|---|---|
| **In Memory Short Term Memory Store** | `ballerina/ai` | No — data lives in the process's memory. | Most agents. Lightweight, fast, no infrastructure. |
| **MSSQL Short Term Memory Store** | `ballerinax/ai.mssql` | Yes — backed by an MS SQL database. | Long-running conversations, audit/compliance, conversations that span sessions. |

Other database-backed stores can be added the same way — the picker grows as new stores are released. For storage backends not in the list (PostgreSQL, MySQL, Redis, …), implement the `ai:Memory` interface in your own class and use that — see [Custom Memory](#custom-memory) below.

The **+ Create New Memory Store** link at the bottom of the picker opens a wizard to add a new memory-store connection to your project (for example, MS SQL with credentials), which then becomes available in the picker for any agent.

## Overflow Configuration

Memory is a sliding window. When new turns push the window over its size limit, the **Overflow Configuration** decides what to do.

| Strategy | Behaviour |
|---|---|
| **Overflow Trim** (default) | Drop the oldest messages, one at a time, until the new turn fits. The most common choice — keeps recent context, forgets the distant past. |
| **(Other strategies)** | Custom configurations can be authored as needed; the field accepts any value the runtime exposes. |

Trim's window length is controlled by the model provider's context window minus a reserve for the system prompt and tool definitions. You don't usually have to tune it yourself.

## Memory Without `ai:Listener`

When the agent is exposed via the `ai:Listener` (the listener the AI Chat Agent Wizard creates), session memory is wired up for you. You don't even need to click **Add Memory** — the listener handles it.

Add explicit memory only when:

- The agent is invoked from somewhere other than `ai:Listener` — for example, a regular `http:Service` or an automation triggered by a Kafka topic.
- You want a non-default store (for example, MSSQL).
- You want to override the overflow strategy.

## Sessions and Isolation

Whichever store you pick, each session ID gets its own memory. If two users hit the same agent at the same time, their conversations stay separate.

```
sessionId = "user-alice-1234"  ────►  Alice's history
sessionId = "user-bob-5678"    ────►  Bob's history
sessionId = "user-alice-9999"  ────►  Alice's *other* conversation
```

The session ID comes in on every chat request (`ai:ChatReqMessage.sessionId`). The listener reads it; you typically don't have to.

## Custom Memory

For stores not in the picker, implement the `ai:Memory` interface in a class of your own. The interface has three methods:

```ballerina
public type Memory distinct isolated object {
    function get(string key) returns ChatMessage[]|MemoryError;
    function update(string key, ChatMessage|ChatMessage[] message) returns MemoryError?;
    function delete(string key) returns MemoryError?;
};
```

A minimal example backed by Postgres:

```ballerina
public isolated class PostgresMemory {
    *ai:Memory;

    private final postgresql:Client db;

    public isolated function init(postgresql:Client db) {
        self.db = db;
    }

    public isolated function get(string key) returns ai:ChatMessage[]|ai:MemoryError {
        // Load messages for `key` from the database and return them.
        return [];
    }

    public isolated function update(string key, ai:ChatMessage|ai:ChatMessage[] message)
            returns ai:MemoryError? {
        // Append messages.
    }

    public isolated function delete(string key) returns ai:MemoryError? {
        // Clear messages for the session.
    }
}
```

Pass an instance into the agent's `memory` field — either through the source view or by registering it as a connection in your project.

## Designing Memory for Your Agent

A short decision tree:

| Situation | Recommended setup |
|---|---|
| New chat agent, just getting started. | Default — In Memory Short Term, default overflow. Don't even click Add Memory. |
| The agent has long conversations and you want to survive restarts. | Add Memory → Short Term Memory → Store: MSSQL Short Term Memory Store. |
| You need conversations to live in your existing PostgreSQL / Redis. | Implement `ai:Memory` (custom). |
| You don't want any context across turns. | Skip the listener — use `agent.run(input)` without a session ID. The agent will treat every call as a fresh conversation. |

## Operational Care

- **Memory contents are sensitive.** Anything the user says is replayed back to the LLM each turn. If conversations may include personal data, treat the memory store like any other personal-data store — encryption at rest, access controls, retention policy.
- **Bound your storage.** A persistent store keeps growing. Add a TTL or a periodic cleanup job for sessions you don't need anymore.
- **Consider compliance.** For regulated industries, memory is a record. The MSSQL store is a good starting point because it's queryable and backupable like any other database table.

## What's Next

- **[Tools](tools.md)** — give the agent capabilities it can apply across remembered turns.
- **[Observability](observability.md)** — see how memory shapes each turn's reasoning.
- **[What is AI Agent Memory?](/docs/genai/key-concepts/what-is-agent-memory)** — conceptual background.
