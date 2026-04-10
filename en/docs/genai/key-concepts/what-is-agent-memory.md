---
sidebar_position: 5
title: What is AI Agent Memory?
description: Understand how AI agents retain conversation context across multiple interactions.
---

# What is AI Agent Memory?

AI Agent memory controls how your AI Agent retains and manages conversation history. Without memory, every message is processed in isolation. With memory, the AI Agent remembers what was said earlier in the conversation, enabling multi-turn interactions like follow-up questions and contextual references.

## Why Memory Matters

Consider this conversation:

```
User: "What's the status of order ORD-001?"
Agent: "ORD-001 has been shipped and arrives March 15."

User: "And ORD-002?"
Agent: ???
```

Without memory, the AI Agent has no context for "And ORD-002?" -- it does not know the user was previously asking about order status. With memory, the AI Agent understands this is a follow-up request for another order's status.

## How Memory Works in WSO2 Integrator

Conversation history is tracked per **session ID**. Each session is an independent thread of messages; different users or conversations never share context.

The recommended way to get memory is to expose your AI Agent over an `ai:Listener` chat service. The listener provides per-session memory automatically -- you do not need to allocate or wire up a memory instance yourself.

```ballerina
import ballerina/ai;

service /support on new ai:Listener(8080) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        // Conversation history for request.sessionId is retained automatically.
        string response = check myAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

For standalone AI Agents that are not served over `ai:Listener`, you can attach an `ai:ShortTermMemory` instance directly.

```ballerina
import ballerina/ai;

final ai:Agent myAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: string `You are a helpful assistant.`
    },
    tools = [lookupOrder, searchProducts],
    memory = new ai:ShortTermMemory(),
    model = check ai:getDefaultModelProvider()
);
```

## Session Isolation

Each session ID creates an independent conversation thread. Different users or conversations never share context.

```ballerina
// Independent conversations -- each session has its own history.
string r1 = check myAgent.run("Hello!", "user-alice");
string r2 = check myAgent.run("Hello!", "user-bob");
string r3 = check myAgent.run("Follow up", "user-alice");  // Only sees Alice's history
```

## Choosing the Right Memory Strategy

| Scenario | Recommended Approach |
|----------|----------------------|
| Chat service with per-session history | Use `ai:Listener` -- memory is automatic |
| Standalone AI Agent with short conversations | `ai:ShortTermMemory` |
| Single-turn task processing | No memory -- call `agent.run(input)` without a session ID |

## What's Next

- [What is MCP?](what-is-mcp.md) -- Model Context Protocol for external AI access
- [What is RAG?](what-is-rag.md) -- Retrieval-augmented generation
- [Adding Memory to an Agent](/docs/genai/develop/agents/adding-memory) -- Implementation guide
