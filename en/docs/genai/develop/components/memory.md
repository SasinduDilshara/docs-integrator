---
sidebar_position: 7
title: AI Agent Memory
description: Reference for every AI Agent Memory implementation in WSO2 Integrator BI — the default Short Term Memory (in-memory store) and the durable MS SQL store, with the BI Configure Memory panel, store picker, and per-store create forms.
---

# AI Agent Memory

**AI Agent Memory** stores per-session chat history for an Agent. Without memory, every agent turn is independent — the agent has no idea what the user said earlier in the conversation. With memory, the agent reasons across the full session.

> Memory is **only** used by Agents. Direct LLM calls, natural functions, and RAG retrieval do not touch memory.

## Public Actions

The Memory contract is small. You rarely call these directly — the Agent runtime handles them for you when an agent is invoked on `ai:Listener`.

| Action | What it does | Required parameters |
|---|---|---|
| **Get** | Returns every stored chat message for the session. | **Session ID**. |
| **Update** | Appends one or more chat messages to the session. The implementation handles overflow (trim or summarise). | **Session ID**, **Message** (one or more chat messages). |
| **Delete** | Drops every stored message for the session. | **Session ID**. |

A "chat message" is one of:

| Role | What it represents |
|---|---|
| `SYSTEM` | The system prompt (typically once per session). |
| `USER` | A user turn. |
| `ASSISTANT` | An agent turn, including any tool calls it decided to make. |
| `FUNCTION` | The result of a tool call returned to the agent. |

## Implementations at a Glance

| Implementation | Status | Durability |
|---|---|---|
| **Short Term Memory** | Default | In-memory (process-local) by default; pluggable backing store. |
| **MS SQL Short Term Memory Store** | Available | Durable (table on Microsoft SQL Server). |
| **PostgreSQL Short Term Memory Store** | Planned | — |
| **Redis Short Term Memory Store** | Planned | — |

---

## Where Memory Is Configured in BI

Memory is wired in from inside an **AI Agent** node — open an agent, hover the cog at the top-right of the AI Agent node, and click **Configure Memory**. The **Configure Memory** panel slides in from the right.

![Configure Memory panel open on the right of an AI Chat Agent flow. Select Memory dropdown shows "Short Term Memory" with an inline description ("Initializes short term memory with an optional store and overflow configuration"). Memory Name field is set to "aiShorttermmemory". A Save button is at the bottom right.](/img/genai/develop/components/memory/01-configure-memory-panel.png)

| Field | What it does |
|---|---|
| **Select Memory** | Pick a memory implementation from the available list. The default is **Short Term Memory**. |
| **Memory Name** | Variable name for this memory instance, used in the generated source. |

Click **Advanced Configurations → Expand** to reveal the **Store** and **Overflow Configuration** rows.

![The same Configure Memory panel with Advanced Configurations expanded. A Store row shows the default ("In-Memory Short Term Memory Store") with a "+ Create New Memory Store" link below it. An Overflow Configuration row shows the default Overflow Trim Configuration (Trim Count 1).](/img/genai/develop/components/memory/02-configure-memory-advanced.png)

| Advanced field | Default | What it controls |
|---|---|---|
| **Store** | In-Memory Short Term Memory Store | The backing storage. Defaults to a process-local in-memory store; switch to **MS SQL** for durability across restarts. |
| **Overflow Configuration** | Trim Overflow Handler (Trim Count `1`) | What happens when the message store fills up. See [Overflow Configurations](#overflow-configurations). |

---

## Picking a Store

To use a different store (or to tune the in-memory store), click **+ Create New Memory Store** under the **Store** row.

![Configure Memory panel with the cursor over the "+ Create New Memory Store" link beneath the Store row.](/img/genai/develop/components/memory/03-create-new-memory-store-link.png)

The **Select Memory Store** picker opens with every store the project knows about.

![Select Memory Store picker showing two entries: "In-Memory Short Term Memory Store" (Provides an in-memory chat message store) and "MSSQL Short Term Memory Store" (Represents an MS SQL backed short term memory store for messages).](/img/genai/develop/components/memory/04-select-memory-store.png)

| Store | When to use |
|---|---|
| **In-Memory Short Term Memory Store** | Single-instance development; tests; short-lived sessions. Process-local — does not survive restarts. |
| **MSSQL Short Term Memory Store** | Production; multi-replica deployments; sessions that must survive restarts. Durable. |

---

## Short Term Memory (Default)

The default memory implementation. Backed by a message store and an overflow strategy that decides what to do when the store fills up.

### In-Memory Store — Basic

Pick **In-Memory Short Term Memory Store** in the picker. The basic create form has only the universal **Memory Store Name** and **Result Type** fields.

![Create Memory Store form for In-Memory Short Term Memory Store with Memory Store Name "aiInmemoryshorttermmemorystore" and Result Type "ai:InMemoryShortTermMemoryStore" (locked). A note above reads "This operation has no required parameters. Optional settings can be configured below."](/img/genai/develop/components/memory/05-create-inmemory-store-basic.png)

### In-Memory Store — Advanced

Click **Advanced Configurations → Expand** to set the per-session capacity.

![Same form with Advanced Configurations expanded. A Size field shows the default (10) and a description: "The maximum capacity for stored messages". The current value in the input is 8.](/img/genai/develop/components/memory/06-create-inmemory-store-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Size** | `10` | Any integer ≥ 3 | Maximum interactive messages stored per session key. The oldest messages are subject to the overflow handler when the store reaches this capacity. |

### Overflow Configurations

When a session reaches the configured **Size**, the overflow handler decides what gets dropped or replaced.

#### Trim Overflow Handler *(default)*

Drops the oldest interactive messages.

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Trim Count** | `1` | Any positive integer | How many messages to remove on each overflow event. |

#### Model Assisted Overflow Handler

Asks an LLM to summarise the oldest messages and replaces them with the summary. Preserves context that trimming would discard, at the cost of an extra LLM call.

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Model** | `()` (uses the default WSO2 model) | Any saved [Model Provider](/docs/genai/develop/components/model-providers) | The model used to produce the summary. |
| **Prompt** | A built-in summarisation prompt | Any prompt template | The summarisation prompt. |

---

## MS SQL Store

A durable Short Term Memory Store backed by Microsoft SQL Server. The connector creates the message table automatically on first use. Official: [Microsoft SQL Server](https://www.microsoft.com/sql-server).

### MS SQL Store — Basic

Pick **MSSQL Short Term Memory Store** in the picker. The basic create form requires an **MS SQL Client** (or a database configuration record).

![Create Memory Store form for MSSQL Short Term Memory Store. MS SQL Client field shows a Record/Expression toggle and a connection record placeholder. Memory Store Name is "mssqlShorttermmemorystore" and Result Type is "mssql:ShortTermMemoryStore" (locked).](/img/genai/develop/components/memory/07-create-mssql-store-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **MS SQL Client** | Yes | — | An existing `mssql:Client` connection, **or** a database configuration record (host, user, password, database, port, instance, options, connection pool). |

### MS SQL Store — Advanced

Click **Advanced Configurations → Expand** to tune caching, per-key capacity, and the table name.

![Same form with Advanced Configurations expanded. Cache Config field with Record/Expression toggle. Max Messages Per Key (default 20) currently set to 8. Table Name (default "ChatMessages") with a note about naming rules. Memory Store Name and Result Type at the bottom.](/img/genai/develop/components/memory/08-create-mssql-store-advanced.png)

| Field | Required | Default | What it controls |
|---|---|---|---|
| **Cache Config** | No | `()` | Optional in-memory cache layer in front of MS SQL. Useful for high-traffic agents. |
| **Max Messages Per Key** | No | `20` | Maximum interactive messages stored per session. |
| **Table Name** | No | `"ChatMessages"` | Name of the database table. Must match `^[A-Za-z_][A-Za-z0-9_]*$` — start with a letter or underscore, contain only letters, digits, and underscores. |

### Database Configuration Record

When you pass a database config record (instead of an existing client), the fields are:

| Field | Required | Default | Available values |
|---|---|---|---|
| **Host** | No | `"localhost"` | Database host. |
| **User** | No | `"sa"` | Database user. |
| **Password** | No | `()` | Database password. |
| **Database** | Yes | — | Database name. |
| **Port** | No | `1433` | Database port. |
| **Instance** | No | `()` | SQL Server instance name. |
| **Options** | No | `()` | Additional MSSQL client options. |
| **Connection Pool** | No | `()` | Connection pool settings. |

### Auto-created Schema

The connector creates the table on first init if it doesn't already exist:

| Column | Type | Notes |
|---|---|---|
| `Id` | `INT IDENTITY(1,1) PRIMARY KEY` | Auto-incrementing row id. |
| `MessageKey` | `NVARCHAR(100) NOT NULL` | The session id. |
| `MessageRole` | `NVARCHAR(20) NOT NULL` | One of `'user'`, `'system'`, `'assistant'`, `'function'`. |
| `MessageJson` | `NVARCHAR(MAX) NOT NULL` | The full chat message serialised to JSON. |
| `CreatedAt` | `DATETIME2 NOT NULL DEFAULT SYSDATETIME()` | Insertion timestamp. |

The system message for a key is upserted (one row per key); interactive messages append.

---

## Picking a Memory Backend

| Situation | Recommended |
|---|---|
| Single-instance dev; tests; short-lived sessions | **Short Term Memory** with the default in-memory store. |
| Production agent that must survive restarts | **MS SQL Store**. |
| Conversation may exceed the window | **Short Term Memory** with **Model Assisted Overflow Handler** so older turns are summarised rather than dropped. |
| Multiple integration replicas behind a load balancer | A durable shared store is **required** — every replica must see the same memory. |

## What's Next

- **[Agents](/docs/genai/develop/agents/overview)** — where memory plugs in.
- **[Develop AI Agents → Memory](/docs/genai/develop/agents/memory)** — the BI canvas walkthrough for wiring memory into an agent.
- **[Memory key concept](/docs/genai/key-concepts/what-is-agent-memory)** — the conceptual background.
