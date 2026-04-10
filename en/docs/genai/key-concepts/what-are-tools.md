---
sidebar_position: 4
title: What are Tools?
description: Understand tools -- the bridge between AI agent reasoning and integration logic.
---

# What are Tools?

Tools are Ballerina functions that an AI Agent can call during its reasoning loop. They are the bridge between the LLM's reasoning and your integration logic -- APIs, databases, services, and business rules.

The LLM sees the tool's name, description, and parameter schema, then decides whether and how to call it. The LLM **never** executes code directly; it produces a structured tool call request, and the agent runtime executes the actual function safely.

## How Tools Work

A tool is an `isolated` Ballerina function annotated with `@ai:AgentTool`. The tool name, description, and parameter descriptions are all derived from the function signature and its Ballerina doc comment -- there is no separate metadata to maintain.

```ballerina
import ballerina/ai;

# Get the current weather for a city.
#
# + city - City name, e.g., "San Francisco"
# + return - Temperature, conditions, and humidity for the city
@ai:AgentTool
isolated function getWeather(string city) returns json|error {
    return check weatherApi->get(string `/weather/${city}`);
}
```

When a user asks "What's the weather in Tokyo?", the AI Agent's LLM reads the tool description, decides to call `getWeather` with `city: "Tokyo"`, and the runtime executes the function and returns the result to the LLM for response generation.

## Tool Categories

| Category | Purpose | Example |
|----------|---------|---------|
| **Data retrieval** | Read information from external systems | Look up customer records, search products |
| **Action tools** | Perform write operations or trigger workflows | Create tickets, send notifications |
| **Connector tools** | Wrap existing WSO2 Integrator connectors | Query Salesforce, interact with databases |

## Tool Design Principles

1. **Clear doc comments** -- The function's doc comment becomes the tool description the LLM sees; make it specific about what the tool does and when to use it
2. **Describe every parameter** -- Use `# + name - description` lines for each parameter so the LLM knows how to fill them in
3. **Keep functions `isolated`** -- Every `@ai:AgentTool` must be an isolated function
4. **Informative errors** -- Return descriptive error messages so the LLM can reason about failures
5. **Limited output** -- Trim large responses to prevent exceeding context window limits

## Registering Tools with an Agent

Pass the annotated functions to the `tools` field when constructing the AI Agent.

```ballerina
import ballerina/ai;

final ai:Agent myAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: string `You are a helpful customer support assistant.`
    },
    tools = [getCustomerDetails, searchOrders, createTicket],
    model = check ai:getDefaultModelProvider()
);
```

## What's Next

- [What is AI Agent Memory?](what-is-agent-memory.md) -- How AI Agents maintain context
- [Adding Tools to an Agent](/docs/genai/develop/agents/adding-tools) -- Detailed tool patterns
- [What is MCP?](what-is-mcp.md) -- Expose tools to external AI assistants
