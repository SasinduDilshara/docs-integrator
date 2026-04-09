---
title: "Quick Start: Set Up WSO2 Integrator"
description: Install and configure WSO2 Integrator to start building AI-powered integrations.
---

# Quick Start: Set Up WSO2 Integrator

**Time:** 10-15 minutes | **What you'll do:** Install WSO2 Integrator, set up your development environment, and verify everything works by creating your first GenAI project.

This guide walks you through getting WSO2 Integrator ready for building AI-powered integrations -- from installation through creating and running your first AI agent project.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An API key for an LLM provider (OpenAI, Azure OpenAI, or Anthropic)

:::info
This quick start uses OpenAI as the model provider. You can swap in any supported provider -- see [AI & LLM Connectors](/docs/connectors/ai-llms) for connection configuration details.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `weather-assistant` and choose a directory.

## Step 2: Add an Agent Artifact

1. In the project explorer, click **+ Add Artifact**.
2. Select **Chat Agent** as the artifact type.
3. Name the agent `WeatherAssistant`.

WSO2 Integrator generates the following scaffold:

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

final agent:ChatAgent weatherAssistant = check new (
    systemPrompt = "You are a helpful weather and calendar assistant.",
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20)
);
```

## Step 3: Configure the System Prompt

Update the `systemPrompt` to give the agent a clear role and behavioral guidelines:

```ballerina
final agent:ChatAgent weatherAssistant = check new (
    systemPrompt = string `You are a personal assistant that helps with weather and calendar queries.
        Guidelines:
        - Always confirm the city or date before fetching data.
        - Respond concisely and use bullet points for lists.
        - If you don't have a tool for something, say so honestly.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20)
);
```

## Step 4: Define and Bind Tools

Add two tools the agent can call -- one for weather data and one for calendar events:

```ballerina
@agent:Tool
isolated function getWeather(string city) returns string|error {
    // In production, call a real weather connector
    return string `Weather in ${city}: 22°C, partly cloudy, humidity 65%`;
}

@agent:Tool
isolated function getCalendarEvents(string date) returns string|error {
    // In production, connect to Google Calendar or Outlook
    return string `Events on ${date}: [10:00 AM] Team standup, [2:00 PM] Design review`;
}
```

Bind the tools to the agent:

```ballerina
final agent:ChatAgent weatherAssistant = check new (
    systemPrompt = "You are a personal assistant that helps with weather and calendar queries.",
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [getWeather, getCalendarEvents]
);
```

## Step 5: Configure the Model Provider

Add your API key to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
```

:::warning
Never commit API keys to version control. Use environment variables or a secrets manager for production deployments. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption) for guidance.
:::

## Step 6: Test with the Chat Interface

1. Click **Run** in the VS Code toolbar.
2. WSO2 Integrator launches the built-in chat interface automatically.
3. Try these messages:

```
You: What's the weather in London?
Agent: The weather in London is 22°C, partly cloudy with 65% humidity.

You: Do I have any meetings tomorrow?
Agent: Here are your events for tomorrow:
  - 10:00 AM — Team standup
  - 2:00 PM — Design review
```

:::tip
Open the **Trace** panel to inspect the full agent loop -- you can see each tool call, the LLM reasoning steps, and token usage in real time.
:::

## Complete Code

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

@agent:Tool
isolated function getWeather(string city) returns string|error {
    return string `Weather in ${city}: 22°C, partly cloudy, humidity 65%`;
}

@agent:Tool
isolated function getCalendarEvents(string date) returns string|error {
    return string `Events on ${date}: [10:00 AM] Team standup, [2:00 PM] Design review`;
}

final agent:ChatAgent weatherAssistant = check new (
    systemPrompt = string `You are a personal assistant that helps with weather and calendar queries.
        Guidelines:
        - Always confirm the city or date before fetching data.
        - Respond concisely and use bullet points for lists.
        - If you don't have a tool for something, say so honestly.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [getWeather, getCalendarEvents]
);
```

## What's Next

- [Build a Smart Calculator Assistant](build-your-first-ai-integration/smart-calculator-assistant.md) -- A step-by-step tutorial for building a math tutor agent with tool binding and tracing
- [What is an AI Agent?](../key-concepts/what-is-an-ai-agent.md) -- Understand the agent loop, memory types, and how agents reason
- [Adding Tools to an Agent](../build-ai-applications/ai-agents/adding-tools-to-an-agent.md) -- Connect agents to real connectors, APIs, and external services
- [Adding Memory to an Agent](../build-ai-applications/ai-agents/adding-memory-to-an-agent.md) -- Choose the right memory strategy for your use case
