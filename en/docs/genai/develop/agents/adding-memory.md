---
sidebar_position: 3
title: Adding Memory to an Agent
description: Configure agent memory for conversation persistence using built-in and custom memory implementations.
---

# Adding Memory to an Agent

Agent memory controls how your AI agent retains conversation history across turns. Without memory, every call is independent and the agent has no awareness of previous interactions. With memory, the agent can refer back to earlier messages and build on prior reasoning.

The WSO2 Integrator `ai` module gives you two primary options:

1. **Session memory via `ai:Listener`** -- the recommended default. Host the agent on `ai:Listener` and pass the `sessionId` from each request into `agent.run(...)`. The listener manages per-session history for you.
2. **Explicit memory on the agent** -- pass a `memory` value to the agent constructor. Use this for standalone agents or when you need a custom persistence strategy.

## Session Memory via `ai:Listener`

This is the simplest and most common setup. The listener provides a built-in short-term memory per session.

```ballerina
import ballerina/ai;

service /tasks on new ai:Listener(8080) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        string response = check taskAssistantAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

- `ai:ChatReqMessage` is `{ string sessionId; string message; }`
- `ai:ChatRespMessage` is `{ string message; }`

Each unique `sessionId` creates an independent conversation. You do not need to configure memory on the agent itself in this mode -- the listener handles it.

## Explicit Short-Term Memory

For a standalone agent (for example, one driven from `main()` or a non-chat service), pass an `ai:ShortTermMemory` instance as the `memory` field.

```ballerina
import ballerina/ai;

final ai:Agent helpDeskAgent = check new (
    systemPrompt = {
        role: "IT Support",
        instructions: "You are a helpful IT support assistant."
    },
    tools = [resetPassword, checkVpnStatus],
    model = check ai:getDefaultModelProvider(),
    memory = new ai:ShortTermMemory()
);
```

`ai:ShortTermMemory` keeps a sliding window of recent messages. It is the right choice for interactive agents where only the most recent exchanges matter and older context can be discarded.

**Best for:** General-purpose chat agents, help desk bots, and interactive exploration where conversations are relatively short.

## Custom Persistent Memory

For conversations that must survive restarts -- or that you need to persist in a database for audit or analytics -- implement the `ai:Memory` interface in your own class and pass an instance to the agent.

The interface looks like this:

```ballerina
public type Memory distinct isolated object {
    function get(string key) returns ChatMessage[]|MemoryError;
    function update(string key, ChatMessage|ChatMessage[] message) returns MemoryError?;
    function delete(string key) returns MemoryError?;
};
```

A minimal PostgreSQL-backed implementation might look like:

```ballerina
import ballerina/ai;
import ballerinax/postgresql;

public isolated class PostgresMemory {
    *ai:Memory;

    private final postgresql:Client db;

    public isolated function init(postgresql:Client db) {
        self.db = db;
    }

    public isolated function get(string key) returns ai:ChatMessage[]|ai:MemoryError {
        // Load messages for the session from the database and return them.
        // Convert rows to ai:ChatMessage values before returning.
        return [];
    }

    public isolated function update(string key, ai:ChatMessage|ai:ChatMessage[] message)
            returns ai:MemoryError? {
        // Append the new message(s) to the session's row(s).
    }

    public isolated function delete(string key) returns ai:MemoryError? {
        // Remove all messages for the session.
    }
}
```

Wire it into the agent:

```ballerina
final postgresql:Client pgClient = check new (
    host = dbHost, database = dbName,
    username = dbUser, password = dbPassword
);

final ai:Agent auditableAgent = check new (
    systemPrompt = {
        role: "Healthcare Scheduler",
        instructions: "You help patients book appointments."
    },
    tools = [searchSlots, bookAppointment],
    model = check ai:getDefaultModelProvider(),
    memory = new PostgresMemory(pgClient)
);
```

**Best for:** Multi-day workflows, compliance and audit use cases, and environments where conversation history must survive restarts.

## Combining Memory with Context Injection

You can add external context to a single turn by prefixing it onto the user message. It will become part of that turn's message and will be stored in memory like any other message.

```ballerina
import ballerina/ai;

service /agent on new ai:Listener(8090) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        json userContext = check getUserProfile(request.sessionId);

        string contextualMessage = string `[User Context: ${userContext.toString()}]

            User question: ${request.message}`;

        string response = check helpDeskAgent.run(contextualMessage, request.sessionId);
        return {message: response};
    }
}
```

If you want context that does NOT persist across turns, inject it on every request rather than storing it in memory.

## Choosing the Right Memory Strategy

| Scenario | Recommended Approach |
|----------|---------------------|
| Typical chat service | Host the agent on `ai:Listener` (built-in session memory) |
| Standalone agent driven from `main()` | `memory = new ai:ShortTermMemory()` |
| Conversations that must survive restarts | Custom class implementing `ai:Memory`, backed by a database |
| Single-turn task processing | Omit memory entirely -- each call is independent |
| Audit or compliance requirements | Custom `ai:Memory` backed by your system of record |

## What's Next

- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build your first agent
- [Adding Tools](/docs/genai/develop/agents/adding-tools) -- Connect agents to functions and APIs
- [Advanced Configuration](/docs/genai/develop/agents/advanced-config) -- Multi-agent orchestration and advanced settings
- [AI Agent Observability](/docs/genai/develop/agents/agent-observability) -- Monitor agent conversations
