---
sidebar_position: 7
title: Memory
description: Reference for every Memory implementation in WSO2 Integrator BI — the default Short Term Memory (in-memory store) and the durable MS SQL store, with init form fields, defaults, and overflow strategies.
---

# Memory

**Memory** stores per-session chat history for an Agent. Without memory, every agent turn is independent — the agent has no idea what the user said in previous turns. With memory, the agent reasons across the conversation.

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

| Implementation | Module | Status | Durability |
|---|---|---|---|
| **Short Term Memory** | `ballerina/ai` | Default | In-memory (process-local) by default; pluggable backing store. |
| **MS SQL Short Term Memory Store** | [`ballerinax/ai.memory.mssql`](https://central.ballerina.io/ballerinax/ai.memory.mssql/latest) | Available | Durable (table on Microsoft SQL Server). |
| **PostgreSQL Short Term Memory Store** | [`ballerinax/ai.memory.postgresql`](https://github.com/ballerina-platform/module-ballerinax-ai.memory.postgresql) | Planned (placeholder repo) | — |
| **Redis Short Term Memory Store** | [`ballerinax/ai.memory.redis`](https://github.com/ballerina-platform/module-ballerinax-ai.memory.redis) | Planned (placeholder repo) | — |

> The Vector Knowledge Base, Chunker, and Embedding Provider sections of this site have BI screenshots; the **Configure Memory** screenshot lives in the [Agents → Memory](/docs/genai/develop/agents/memory) page since memory is only configured from inside the AI Agent node. This page is the reference for what each option does.

---

## Short Term Memory (Default)

The default memory implementation. Backed by an in-memory message store and an overflow strategy that decides what to do when the store fills up.

### Configuration fields

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Store** | In-Memory Short Term Memory Store | An in-memory store, the MS SQL store, or any custom Short Term Memory Store | The backing storage. Defaults to a process-local store; switch to MS SQL for durability across restarts. See [Stores](#stores) below. |
| **Overflow Configuration** | Trim Overflow Handler | Trim Overflow Handler, Model Assisted Overflow Handler | What happens when the message store fills up. See [Overflow Configurations](#overflow-configurations) below. |

### Stores

The store is what actually persists messages. Pick one from the table below.

| Store | When to use | Configuration |
|---|---|---|
| **In-Memory Short Term Memory Store** | Single-instance development; tests; short-lived sessions. | Optional **Size** (default `10`, must be ≥ 3). Maximum interactive messages stored per session key. |
| **MS SQL Short Term Memory Store** | Production; multi-replica deployments; sessions that must survive restarts. | See [MS SQL Store](#ms-sql-store) below. |

### Overflow Configurations

When the store reaches capacity for a session, the overflow handler decides what gets dropped or replaced.

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

### Init form fields

| Field | Required | Default | Available values |
|---|---|---|---|
| **MSSQL Client** | Yes | — | An existing `mssql:Client` connection, **or** a database configuration record (host, user, password, database, port, instance, options, connection pool). |
| **Max Messages Per Key** | No | `20` | Maximum interactive messages stored per session. |
| **Cache Configuration** | No | `()` | Optional in-memory cache layer. Useful for high-traffic agents. |
| **Table Name** | No | `"ChatMessages"` | Name of the database table. Must match `^[A-Za-z_][A-Za-z0-9_]*$`. |

### Database configuration record

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

### Auto-created schema

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
