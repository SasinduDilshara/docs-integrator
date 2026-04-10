---
sidebar_position: 1
title: Creating an AI Agent
description: Build interactive chat agents and task agents with system prompts, tools, and memory.
---

# Creating an AI Agent

AI agents are conversational components that combine an LLM's reasoning with tools and memory to solve user tasks over multiple turns. In WSO2 Integrator you build them with the Ballerina `ai` module -- everything you need is in `ballerina/ai`, and provider-specific connectors live under `ballerinax/ai.*`.

## Creating a Basic Agent

The simplest agent needs three things: a system prompt, one or more tools, and a model provider.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:Agent taskAssistantAgent = check new (
    systemPrompt = {
        role: "Task Assistant",
        instructions: string `You are a helpful assistant for managing
            a to-do list. You can manage tasks and help a user plan
            their schedule.`
    },
    tools = [addTask, listTasks, getCurrentDate],
    model = check ai:getDefaultModelProvider()
);

public function main() returns error? {
    while true {
        string userInput = io:readln("User (or 'exit' to quit): ");
        if userInput == "exit" {
            break;
        }
        string response = check taskAssistantAgent.run(userInput);
        io:println("Agent: ", response);
    }
}
```

`ai:getDefaultModelProvider()` reads the model configuration from your `Config.toml`, so you can switch providers without touching the code. If you prefer to wire the provider explicitly, import the relevant connector (for example, `ballerinax/ai.openai`, `ballerinax/ai.anthropic`, or `ballerinax/ai.azure`) and construct the provider directly.

## System Prompt Design

The system prompt defines the agent's role and the rules it must follow. In the `ai` module it is a record with a short `role` and a longer `instructions` string.

```ballerina
ai:SystemPrompt analystPrompt = {
    role: "Financial Analyst",
    instructions: string `You are a financial analyst assistant for Acme Corp.

        Role and Scope:
        - Help employees analyze quarterly financial data.
        - Generate summaries of revenue, expenses, and profit trends.

        Rules:
        - Never reveal raw database queries or internal system details.
        - Round all currency values to two decimal places.
        - If asked about topics outside financial analysis, politely redirect.

        Response Format:
        - Use bullet points for lists of data points.
        - Include percentage changes when comparing periods.
        - Summarize key takeaways at the end of each analysis.`
};
```

### System Prompt Best Practices

| Practice | Example |
|----------|---------|
| Define the role clearly | "You are a customer support agent for an e-commerce platform" |
| Set boundaries | "Only answer questions about orders, returns, and products" |
| Specify output format | "Always respond with a brief summary followed by details" |
| Include constraints | "Never share customer personal data in responses" |
| Add tool usage guidance | "Use the orderLookup tool before answering order questions" |

## Equivalent Configuration Styles

The agent constructor accepts its configuration either as named arguments or as a single `ai:AgentConfiguration` record. Both forms are equivalent.

```ballerina
final ai:Agent taskAssistantAgent = check new ({
    systemPrompt: {
        role: "Task Assistant",
        instructions: "You are a helpful assistant for managing a to-do list."
    },
    tools: [addTask, listTasks, getCurrentDate],
    model: check ai:getDefaultModelProvider(),
    maxIter: 10
});
```

The most commonly used fields on `ai:AgentConfiguration` are:

| Field | Type | Description |
|-------|------|-------------|
| `systemPrompt` | `ai:SystemPrompt` | Role and instructions given to the LLM |
| `tools` | `(ai:FunctionTool \| ai:BaseToolKit)[]` | Tool functions and toolkits the agent may call |
| `model` | `ai:ModelProvider` | LLM provider that powers the agent |
| `memory` | `ai:Memory?` | Optional explicit memory; defaults to session memory under `ai:Listener` |
| `maxIter` | `int` | Maximum ReAct iterations per query (default `5`) |
| `agentType` | `ai:AgentType` | Reserved for future agent strategies; defaults to `ai:REACT_AGENT` |

## Multi-Turn Conversations

An agent keeps context across turns through its memory component. The easiest way to run a chat service is to expose the agent over `ai:Listener`, which wires session memory automatically using the `sessionId` on each request.

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

`ai:ChatReqMessage` is `{ string sessionId; string message; }` and `ai:ChatRespMessage` is `{ string message; }`. You never have to define your own chat record types -- the module provides them.

## Configuring Agent Behavior

### Maximum Iterations

Use `maxIter` to cap how many reason-act-observe loops the agent can run per query. If the agent reaches the limit it returns the best answer it has so far.

```ballerina
final ai:Agent boundedAgent = check new (
    systemPrompt = {
        role: "Research Assistant",
        instructions: "You are a research assistant."
    },
    tools = [searchDatabase, fetchDocument],
    model = check ai:getDefaultModelProvider(),
    maxIter = 5
);
```

### Tuning the Model Provider

Model-specific parameters such as `temperature` or `topP` belong on the provider, not on the agent. For example, when you construct an OpenAI provider directly you can pass model parameters to its constructor (see the provider documentation). The agent then uses that provider without any further changes.

## Session Isolation

Each unique `sessionId` creates an independent conversation thread when the agent is hosted on an `ai:Listener`.

```ballerina
string r1 = check taskAssistantAgent.run("Hello!", "user-alice-session-1");
string r2 = check taskAssistantAgent.run("Hello!", "user-bob-session-1");
string r3 = check taskAssistantAgent.run("Follow up", "user-alice-session-1");
// r3 only sees Alice's history.
```

## Handling Errors in Tools

Tools can return errors, and the agent will surface them back to the LLM so the model can reason about the failure. Return a descriptive error or a structured record so the agent can communicate the issue to the user.

```ballerina
import ballerina/ai;

# Returns the current balance for a bank account.
# + accountId - The customer's account identifier
# + return - Current balance or an error explaining the failure
@ai:AgentTool
isolated function getAccountBalance(string accountId) returns json|error {
    json|error result = bankApi->/accounts/[accountId]/balance;
    if result is error {
        return error(string `Unable to retrieve balance for account ${accountId}. ` +
            "The banking system may be temporarily unavailable.");
    }
    return result;
}
```

## What's Next

- [Adding Tools](/docs/genai/develop/agents/adding-tools) -- Connect agents to functions and APIs
- [Adding Memory](/docs/genai/develop/agents/adding-memory) -- Configure conversation history
- [Advanced Configuration](/docs/genai/develop/agents/advanced-config) -- Multi-agent orchestration and API exposure
