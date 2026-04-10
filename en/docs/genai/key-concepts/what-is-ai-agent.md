---
sidebar_position: 3
title: What is an AI Agent?
description: Understand AI agents -- autonomous components that combine LLMs with tools, memory, and a reasoning loop.
---

# What is an AI Agent?

An AI Agent is an integration component that combines an LLM with tools, memory, and a reasoning loop. Unlike a simple LLM API call, an AI Agent can reason about what steps to take, call tools to gather data or perform actions, observe the results, and decide what to do next.

## The Agent Loop

Every AI Agent in WSO2 Integrator follows the **Reason-Act-Observe** loop:

```
User Message
    |
    v
+-----------+
|  Reason   | <-- LLM analyzes the message + context + tool descriptions
+-----------+
    |
    v
+-----------+
|   Act     | <-- Agent calls one or more tools based on the LLM's decision
+-----------+
    |
    v
+-----------+
|  Observe  | <-- Tool results are fed back to the LLM
+-----------+
    |
    v
Final Response (or loop again)
```

This loop can repeat multiple times per request. For example, an AI Agent might look up a customer record (first tool call), then check their order history (second tool call), and finally compose a response using both pieces of information.

## Agent Components

An AI Agent has four components: a model, a system prompt, tools, and memory.

```ballerina
import ballerina/ai;

final ai:Agent myAgent = check new (
    // 1. System prompt -- defines the agent's role and behavior.
    systemPrompt = {
        role: "Support Assistant",
        instructions: string `You are a helpful customer support assistant.`
    },

    // 2. Tools -- functions the agent can call.
    tools: [lookupCustomer, createOrder, sendEmail],

    // 3. Model -- the LLM that powers reasoning.
    model = check ai:getDefaultModelProvider()
);
```

| Component | Purpose | Required |
|-----------|---------|----------|
| **Model** | LLM connection for reasoning and response generation | Yes |
| **System Prompt** | Defines the AI Agent's role, personality, and constraints | Yes |
| **Tools** | Functions the AI Agent can call during reasoning | No (but recommended) |
| **Memory** | Stores conversation history for multi-turn interactions | No (provided automatically when using `ai:Listener`) |

## Running an Agent

You invoke an AI Agent by calling `run` with the user's input. If you want session-scoped conversation memory, pass a session ID as the second argument.

```ballerina
// Single-shot invocation.
string response = check myAgent.run("What's the status of order ORD-001?");

// With session memory -- messages for the same session ID share context.
string response = check myAgent.run("And what about ORD-002?", "session-123");
```

When you expose an AI Agent over an `ai:Listener` chat service, session memory is provided automatically for each `sessionId` in the request.

```ballerina
service /support on new ai:Listener(8080) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        string response = check myAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

## What's Next

- [What are Tools?](what-are-tools.md) -- Understand how AI Agents interact with external systems
- [What is AI Agent Memory?](what-is-agent-memory.md) -- How AI Agents maintain conversation context
- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build your first AI Agent
