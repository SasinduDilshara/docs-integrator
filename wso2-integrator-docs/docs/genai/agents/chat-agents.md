---
sidebar_position: 2
title: "Create a Chat Agent"
description: Build a user-facing conversational agent with a system prompt, LLM connection, multi-turn memory, and streaming responses.
---

# Create a Chat Agent

Chat agents are the primary way to build user-facing conversational experiences in WSO2 Integrator. You define a role, connect an LLM, configure memory, and bind tools — then WSO2 Integrator handles the REST endpoint, session management, and chat interface for you.

This guide walks you through creating a chat agent both visually and in pro-code, using a math tutor as the running example.

## Create a Chat Agent in the Visual Designer

1. Open your project in VS Code.
2. Click **+ Add Artifact** and select **Chat Agent**.
3. Name the agent `MathTutor`.
4. The visual designer opens with the agent configuration panel.

Configure the following in the properties panel:

| Property | Value |
|---|---|
| **System Prompt** | "You are a patient math tutor..." |
| **Model Provider** | OpenAI (gpt-4o) |
| **Memory Type** | Message Window (20 messages) |
| **Tools** | solveEquation, plotGraph |

## Create a Chat Agent in Pro-Code

Switch to the pro-code view to see the generated Ballerina code, or write it directly:

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

@agent:Tool
isolated function solveEquation(string equation) returns string|error {
    // Parse and solve the equation
    return string `Solution for ${equation}: x = 42`;
}

@agent:Tool
isolated function plotGraph(string expression, float rangeStart, float rangeEnd) returns string|error {
    // Generate a graph URL
    return string `Graph plotted for ${expression} from ${rangeStart} to ${rangeEnd}`;
}

final agent:ChatAgent mathTutor = check new (
    systemPrompt = string `You are a patient math tutor who explains concepts step by step.
        Rules:
        - Break down problems into small steps.
        - Use simple language, avoid jargon.
        - Offer to solve or plot when appropriate.
        - Always encourage the student.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [solveEquation, plotGraph]
);
```

:::info
For LLM connection configuration details (API keys, endpoints, model selection), see [AI & LLM Connectors](/docs/connectors/ai-llms).
:::

## Configure the System Prompt

The system prompt is the most important configuration for your agent. It defines the agent's persona, capabilities, and behavioral constraints.

Effective system prompts include:
- **Role definition** — Who the agent is and what it does
- **Behavioral guidelines** — How it should respond (tone, format, length)
- **Constraints** — What it should not do
- **Tool usage instructions** — When to use specific tools

```ballerina
systemPrompt = string `You are a senior math tutor specializing in algebra and calculus.

    Behavior:
    - Always explain your reasoning step by step.
    - Use LaTeX notation for mathematical expressions.
    - If the student is stuck, offer a hint before the full solution.

    Constraints:
    - Only help with math topics. Politely redirect other questions.
    - Never give answers without showing work.

    Tool usage:
    - Use solveEquation when the student asks you to verify an answer.
    - Use plotGraph when visualizing a function would help understanding.`
```

## Handle Multi-Turn Conversations

Chat agents manage conversation history automatically through the configured memory. Each session maintains its own context, so multiple users can interact with the same agent concurrently.

```ballerina
// The memory keeps the last 20 messages per session
memory = new agent:MessageWindowChatMemory(maxMessages = 20)
```

When the conversation exceeds the window, older messages are dropped. For more sophisticated memory strategies (semantic retrieval, persistent storage), see [Configure Agent Memory](/docs/genai/agents/memory-configuration).

## Enable Streaming Responses

Streaming lets the agent send partial responses as the LLM generates them, which improves perceived responsiveness for users:

```ballerina
final agent:ChatAgent mathTutor = check new (
    systemPrompt = "You are a patient math tutor.",
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [solveEquation, plotGraph],
    streamResponse = true
);
```

The built-in chat interface and the REST endpoint both support streaming via Server-Sent Events (SSE).

## Use the Built-In Chat Interface

When you run a chat agent, WSO2 Integrator launches a chat interface automatically:

1. Click **Run** in the toolbar.
2. The chat interface opens in a side panel.
3. Type a message and observe the agent's reasoning in real time.

:::tip
Enable the **Trace** panel alongside the chat to inspect each step of the agent loop — LLM calls, tool invocations, memory reads, and token usage.
:::

## Switch Model Providers

You can swap the model provider without changing the agent logic. WSO2 Integrator supports multiple providers:

```ballerina
// OpenAI
model = check new openai:Client({auth: {token: openaiApiKey}})

// Azure OpenAI
model = check new azureopenai:Client({
    auth: {apiKey: azureApiKey},
    serviceUrl: "https://my-resource.openai.azure.com"
})

// Anthropic
model = check new anthropic:Client({auth: {apiKey: anthropicApiKey}})
```

See [AI & LLM Connectors](/docs/connectors/ai-llms) for the full list of supported providers and configuration options.

## What's Next

- [Expose an Agent as an API](/docs/genai/agents/api-exposed-agents) — Serve your agent as a programmatic HTTP/REST endpoint
- [Bind Tools to Agents](/docs/genai/agents/tool-binding) — Connect agents to external services and connectors
- [Configure Agent Memory](/docs/genai/agents/memory-configuration) — Choose between conversation, semantic, and persistent memory
