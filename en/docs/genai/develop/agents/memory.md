---
sidebar_position: 4
title: Memory
description: Reference for configuring AI agent memory in WSO2 Integrator — short-term in-memory, MS SQL, custom stores, overflow strategy, and the after-state with memory attached.
---

# Memory

**Memory** is what lets your agent remember earlier turns of a conversation. WSO2 Integrator handles per-session memory automatically when an agent runs on `ai:Listener` — but the **Configure Memory** dialog lets you tune *what* is remembered, *where* it's stored, and *how long* it survives.

## The Configure Memory Panel

Click **+ Add Memory** on the AI Agent node. The **Configure Memory** panel opens on the right:

![The Configure Memory panel — Select Memory dropdown set to Short Term Memory, with description 'Initializes short-term memory with an optional store and overflow configuration.' This operation has no required parameters info banner. Advanced Configurations section with Store (Default: In-Memory Short Term Memory Store), Overflow Configuration (Default: Overflow Trim), Memory Name set to aiShorttermmemory. Save button at the bottom.](/img/genai/develop/agents/09-configure-memory.png)

| Field | Required | What it does |
|---|---|---|
| **Select Memory*** | Yes | Memory **strategy**. The current option is **Short Term Memory**. |
| **Store** | No (under Advanced) | Where messages are persisted. Default: **In-Memory Short Term Memory Store**. |
| **Overflow Configuration** | No (under Advanced) | What happens when the window is full. Default: **Overflow Trim**. |
| **Memory Name*** | Yes | Variable name in the generated source. Default: `aiShorttermmemory`. |

After saving, the AI Agent block on the canvas grows a Memory sub-block — see [The Agent Box After Memory Is Attached](#the-agent-box-after-memory-is-attached).

### Advanced Configurations Expanded

Click **Expand** on the Advanced Configurations section to see Store, Overflow, and Memory Name in detail:

![The Configure Memory panel with Advanced Configurations expanded. Store field with description 'The memory store to use; if not provided, an in-memory store is used.' (Default: In-Memory Short Term Memory Store, Select/Expression toggle). Below the field: 'No items available. Create one below.' and a '+ Create New Memory Store' link. Overflow Configuration field with description 'The strategy to handle overflow; if not provided, trimming is used.' (Default: Overflow Trim Configuration, Record/Expression toggle, default value `{}`). Memory Name* field set to 'aiShorttermmemory'.](/img/genai/develop/agents/19-configure-memory-advanced.png)

| Field | Toggle | Default | What it does |
|---|---|---|---|
| **Store** | Select / Expression | `In-Memory Short Term Memory Store` | The actual storage backend. The dropdown shows existing memory-store connections in the project. **+ Create New Memory Store** opens the picker to create one inline. |
| **Overflow Configuration** | Record / Expression | `Overflow Trim Configuration ({})` | How the runtime handles a full window. See [Overflow Configuration](#overflow-configuration) below. |
| **Memory Name** | text | `aiShorttermmemory` | Variable name. Should match the project's naming convention. |

## Memory Stores

The **Store** field decides where the conversation history lives. Click the field (or the **+ Create New Memory Store** link) to open the **Select Memory Store** picker:

![The Select Memory Store panel listing two stores: In Memory Short Term Memory Store with description 'Provides an in-memory chat message store.' (highlighted as default) and MSSQL Short Term Memory Store with description 'Represents an MS SQL-backed short-term memory store for messages.'](/img/genai/develop/agents/10-select-memory-store.png)

| Store | Module | Survives a restart? | Good for |
|---|---|---|---|
| **In Memory Short Term Memory Store** | `ballerina/ai` | No — data lives in the process's memory. | Most agents. Lightweight, fast, no infrastructure. |
| **MSSQL Short Term Memory Store** | `ballerinax/ai.mssql` | Yes — backed by an MS SQL database. | Long-running conversations, audit/compliance, conversations that span sessions. |

Other database-backed stores can be added the same way — the picker grows as new stores are released. For backends not in the list (PostgreSQL, MySQL, Redis, …), implement the `ai:Memory` interface in your own class — see [Custom Memory](#custom-memory) below.

### Creating an In-Memory Short Term Memory Store

Picking **In Memory Short Term Memory Store** opens the simplest store form:

| Field | What it does |
|---|---|
| **Memory Store Name*** | Variable name. Default: `aiInmemoryshorttermmemorystore`. |
| **Result Type*** | `ai:InMemoryShortTermMemoryStore` (locked). |

There are no other parameters — the in-memory store is configuration-free.

### Creating an MSSQL Short Term Memory Store

Picking **MSSQL Short Term Memory Store** opens a richer form:

![Create Memory Store form for MSSQL. Fields: MS SQL Client* (with description 'The MS SQL client or database configuration to connect to the database.', Record/Expression toggle, default `new ('', (), (), (), 0, '', (), ())`). Advanced Configurations Expand link. Memory Store Name* (default 'mssqlShorttermmemorystore'). Result Type* (default 'mssql:ShortTermMemoryStore', locked). Save button.](/img/genai/develop/agents/20-create-mssql-memory-store.png)

| Field | Required | What it does |
|---|---|---|
| **MS SQL Client*** | Yes | The MS SQL client or database configuration to connect to the database. **Record / Expression** toggle. The default is the constructor for `mssql:Client`: `new ("", (), (), (), 0, "", (), ())` — fill in host, username, password, database, port, etc. |
| **Advanced Configurations** | No | See below. |
| **Memory Store Name*** | Yes | Variable name. Default: `mssqlShorttermmemorystore`. |
| **Result Type*** | Yes | `mssql:ShortTermMemoryStore` (locked). |

#### MSSQL Memory Store Advanced Configurations

Expand **Advanced Configurations** to see persistence-specific knobs:

![Create Memory Store form for MSSQL with Advanced Configurations expanded. Fields visible: Cache Config (Default: {}, description 'The cache configuration for in-memory caching of messages.', Record/Expression). Max Messages Per Key (Default: 20, description 'The maximum number of interactive messages to store per key.', Number/Expression). Table Name (Default: 'ChatMessages', description 'The name of the database table to store chat messages (default 'ChatMessages'). Must start with a letter or underscore and contain only letters, digits, and underscores.', Text/Expression).](/img/genai/develop/agents/21-mssql-memory-advanced.png)

| Field | Default | What it does |
|---|---|---|
| **Cache Config** | `{}` | In-memory cache layer over the SQL store. Reduces round-trips for hot sessions. Record/Expression. |
| **Max Messages Per Key** | `20` | Hard cap on stored messages per session ID. The oldest are dropped when the cap is reached. Lower for cost-sensitive deployments; raise for long, multi-day conversations. |
| **Table Name** | `"ChatMessages"` | The database table that stores messages. Must start with a letter or underscore; letters, digits, and underscores only. Use a per-environment table name if multiple agents share the same database. |

Inside an `ai:Listener`-hosted agent, every chat request's `sessionId` becomes a row key in this table. Inspecting the table directly is a perfectly valid way to debug a conversation.

## Overflow Configuration

Memory is a sliding window. When new turns push the window over its size limit, the **Overflow Configuration** decides what to do.

| Strategy | Behaviour | Field shape |
|---|---|---|
| **Overflow Trim** (default) | Drop the oldest messages, one at a time, until the new turn fits. The most common choice. | `{}` (no parameters) |
| **(Custom)** | Switch to **Expression** and pass a record satisfying `ai:OverflowConfiguration`. | record value |

Trim's effective window length is controlled by the agent's model context window minus a reserve for the system prompt and tool definitions. You don't usually have to tune it yourself.

## The Agent Box After Memory Is Attached

Once memory is configured, the AI Agent block on the canvas shows it as a sub-block — making the wiring visible at a glance:

![The AI Chat Agent canvas after Memory has been attached. The AI Agent block now has an additional inner block 'Memory: ShortTermMemory' between the agent name (AI Agent / stringResult) and the BlogReviewer label.](/img/genai/develop/agents/22-agent-with-memory-attached.png)

The block now contains:

- **AI Agent / `stringResult`** — top label.
- **Memory: ShortTermMemory** — sub-block; click to edit, click `⋮` to remove.
- **BlogReviewer** — agent role label.
- *"Provide specific instructions on how the agent should behave."* — Instructions placeholder.

## Memory Without `ai:Listener`

When the agent is exposed via `ai:Listener` (the listener the AI Chat Agent Wizard creates), session memory is wired up for you. You don't even need to click **Add Memory** — the listener handles it.

Add explicit memory only when:

- The agent is invoked from somewhere other than `ai:Listener` — for example, a regular `http:Service` or an automation triggered by a Kafka topic.
- You want a non-default store (for example, MSSQL).
- You want to override the overflow strategy.
- You want to share one memory store across multiple agents.

## Sessions and Isolation

Whichever store you pick, each session ID gets its own memory. If two users hit the same agent at the same time, their conversations stay separate.

```bash
sessionId = "user-alice-1234"  ────►  Alice's history
sessionId = "user-bob-5678"    ────►  Bob's history
sessionId = "user-alice-9999"  ────►  Alice's *other* conversation
```

The session ID comes in on every chat request (`ai:ChatReqMessage.sessionId`). The listener reads it; you typically don't have to.

## Custom Memory

For stores not in the picker, implement the `ai:Memory` interface in a class of your own. The interface has three methods:

> Implementations live in your own module and import `ballerina/ai`, so reference the types using the `ai:` prefix (`ai:Memory`, `ai:ChatMessage`, `ai:MemoryError`) — shown unprefixed below for the contract itself.

```ballerina
public type Memory distinct isolated object {
    function get(string key) returns ai:ChatMessage[]|ai:MemoryError;
    function update(string key, ai:ChatMessage|ai:ChatMessage[] message) returns ai:MemoryError?;
    function delete(string key) returns ai:MemoryError?;
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

## Editing Memory After It's Attached

To change the memory configuration after creation:

1. On the canvas, click the **Memory** sub-block inside the AI Agent box, or click the connection in the right-side panel.
2. The Configure Memory panel re-opens with the existing values.
3. Adjust and click Save.

To swap the underlying store (e.g. In-Memory → MSSQL):

1. Open Configure Memory → Advanced → Store.
2. Pick a different store, or click **+ Create New Memory Store** to add one.
3. Save.

The agent picks up the change on the next message — there's no restart required for in-IDE iteration.

## Designing Memory for Your Agent

A short decision tree:

| Situation | Recommended setup |
|---|---|
| New chat agent, just getting started. | Default — In-Memory Short Term, default overflow. Don't click Add Memory at all. |
| The agent has long conversations and you want to survive restarts. | Add Memory → Short Term Memory → Store: **MSSQL Short Term Memory Store**. |
| You need conversations to live in your existing PostgreSQL / Redis. | Implement `ai:Memory` (custom). |
| You don't want any context across turns. | Skip the listener — use `agent.run(input)` without a session ID. Each call is fresh. |
| You want to share memory across multiple agents (e.g. router + specialist). | Create one MSSQL store and select it in both agents' Configure Memory panels. |

## Operational Care

- **Memory contents are sensitive.** Anything the user says is replayed back to the LLM each turn. If conversations may include personal data, treat the memory store like any other personal-data store — encryption at rest, access controls, retention policy.
- **Bound your storage.** A persistent store keeps growing. Set **Max Messages Per Key** to a finite value and add a TTL or periodic cleanup job for sessions you don't need anymore.
- **Consider compliance.** For regulated industries, memory is a record. The MSSQL store is a good starting point because it's queryable and backupable like any other database table.
- **Watch table-name collisions.** If you run multiple agents against the same MSSQL database, give each a unique **Table Name** so their messages don't mix.

## What's Next

- **[Tools](tools.md)** — give the agent capabilities it can apply across remembered turns.
- **[Observability](observability.md)** — see how memory shapes each turn's reasoning.
- **[What is AI Agent Memory?](/docs/genai/key-concepts/what-is-agent-memory)** — conceptual background.
