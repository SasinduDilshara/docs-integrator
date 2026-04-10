---
sidebar_position: 4
title: Advanced AI Agent Configurations
description: Tune agent iterations, compose toolkits, orchestrate multiple agents, and expose them over ai:Listener.
---

# Advanced AI Agent Configurations

Once you have a working agent with tools and memory, you can tune its behavior with a few advanced options on `ai:AgentConfiguration`, combine toolkits, and orchestrate multiple agents. This page covers iteration limits, toolkit composition, multi-agent patterns, and how to expose an agent as an API using `ai:Listener`.

## Core Agent Options

### Maximum Iterations

`maxIter` caps how many reason-act-observe loops the agent can run for a single request. The default is `5`. Increase it for complex tasks that require many tool calls; decrease it to fail fast for interactive agents where users expect a quick answer.

```ballerina
import ballerina/ai;

final ai:Agent boundedAgent = check new (
    systemPrompt = {
        role: "Research Assistant",
        instructions: "You are a research assistant."
    },
    tools = [searchDatabase, fetchDocument, analyzeData],
    model = check ai:getDefaultModelProvider(),
    maxIter = 10
);
```

If the agent hits `maxIter` without converging, it returns the best answer it has assembled so far.

### Agent Type

`agentType` defaults to `ai:REACT_AGENT`. The field is reserved so that future agent strategies can be plugged in without breaking existing code. For now, leave it at the default unless documentation for a specific strategy tells you otherwise.

### Model Parameter Tuning

Parameters like `temperature`, `topP`, and other sampling controls live on the **model provider**, not on the agent. To tune them, construct the provider directly and pass it into the agent.

```ballerina
import ballerina/ai;
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:ModelProvider preciseModel = check new openai:ModelProvider(
    openAiApiKey,
    openai:GPT_4O,
    temperature = 0.1
);

final ai:Agent preciseAgent = check new (
    systemPrompt = {
        role: "Data Analyst",
        instructions: "You are a data analyst. Be precise and factual."
    },
    tools = [queryDatabase],
    model = preciseModel
);
```

Check the provider's own documentation for the exact parameter names it accepts.

## Composing Tools and Toolkits

The `tools` field accepts any mixture of function tools, `ai:McpToolKit` instances, and your own `ai:BaseToolKit` classes. This lets you assemble an agent from reusable capability bundles.

```ballerina
import ballerina/ai;

final ai:McpToolKit weatherMcp = check new ("http://localhost:9090/mcp");

final ai:Agent travelAgent = check new (
    systemPrompt = {
        role: "Travel Assistant",
        instructions: "You help users plan trips end-to-end."
    },
    tools = [searchHotels, weatherMcp, new TaskManagerToolkit()],
    model = check ai:getDefaultModelProvider()
);
```

### Custom Toolkits

A toolkit is an `isolated` class that includes `ai:BaseToolKit` and returns its tool configurations from `getTools()`. See [Adding Tools](/docs/genai/develop/agents/adding-tools) for the full pattern; the short version is:

```ballerina
public isolated class BillingToolkit {
    *ai:BaseToolKit;

    public isolated function getTools() returns ai:ToolConfig[] =>
        ai:getToolConfigs([self.getInvoice, self.processRefund]);

    @ai:AgentTool
    isolated function getInvoice(string invoiceId) returns json|error {
        return check billingApi->/invoices/[invoiceId];
    }

    @ai:AgentTool
    isolated function processRefund(string invoiceId, decimal amount) returns json|error {
        return check billingApi->/invoices/[invoiceId]/refund.post({amount});
    }
}
```

## Custom Model Providers

If you need to integrate an LLM that does not have an off-the-shelf connector, implement the `ai:ModelProvider` interface and pass an instance of your class to the agent. This lets you wrap any HTTP-accessible model while keeping the rest of your agent code unchanged.

```ballerina
public isolated class MyCustomProvider {
    *ai:ModelProvider;
    // Implement the required methods from ai:ModelProvider here.
}

final ai:Agent agent = check new (
    systemPrompt = {role: "Assistant", instructions: "..."},
    tools = [],
    model = new MyCustomProvider()
);
```

## Multi-Agent Orchestration

The `ai` module does not require any special orchestration primitives -- each agent is just a `final ai:Agent`, and you can call one agent from inside a tool that belongs to another. This covers router, pipeline, and handoff patterns with the same building blocks.

### Router Agent Pattern

Use a top-level agent to route requests to specialised sub-agents based on intent.

```ballerina
import ballerina/ai;

final ai:Agent billingAgent = check new (
    systemPrompt = {
        role: "Billing Specialist",
        instructions: "You handle billing, invoices, and payment questions."
    },
    tools = [getInvoice, processRefund, getPaymentHistory],
    model = check ai:getDefaultModelProvider()
);

final ai:Agent technicalAgent = check new (
    systemPrompt = {
        role: "Technical Specialist",
        instructions: "You handle API issues, integration errors, and configuration."
    },
    tools = [checkApiStatus, getErrorLogs, getConfiguration],
    model = check ai:getDefaultModelProvider()
);

# Route a conversation to the billing specialist.
# + message - The user's question
# + return - The specialist's response
@ai:AgentTool
isolated function routeToBilling(string message) returns string|error {
    return billingAgent.run(message);
}

# Route a conversation to the technical specialist.
# + message - The user's question
# + return - The specialist's response
@ai:AgentTool
isolated function routeToTechnical(string message) returns string|error {
    return technicalAgent.run(message);
}

final ai:Agent routerAgent = check new (
    systemPrompt = {
        role: "Support Router",
        instructions: string `You are a customer support router.
            Use routeToBilling for payment and invoice questions.
            Use routeToTechnical for API and integration issues.`
    },
    tools = [routeToBilling, routeToTechnical],
    model = check ai:getDefaultModelProvider()
);
```

### Pipeline Agent Pattern

Chain agents sequentially where each agent's output feeds into the next.

```ballerina
function processDocument(string document) returns string|error {
    string extracted = check extractionAgent.run(
        string `Extract the key information from this document:
            ${document}`
    );

    string classified = check classificationAgent.run(
        string `Classify the following extracted information:
            ${extracted}`
    );

    return check outputAgent.run(
        string `Generate a final summary from:
            ${classified}`
    );
}
```

### Agent Handoff

When routing a live conversation from one agent to another, summarise the conversation so far and pass the summary into the next agent as part of its first prompt. Since there is no cross-agent shared memory API, composing via prompts keeps the pattern simple and predictable.

```ballerina
function handoff(string sessionId, string reason, string conversationSummary)
        returns string|error {
    string seed = string `This conversation has been transferred. Reason: ${reason}.
        Previous conversation summary: ${conversationSummary}.
        Please continue assisting the customer from where the previous agent left off.`;
    return check specialistAgent.run(seed, sessionId);
}
```

## Exposing Agents over `ai:Listener`

The idiomatic way to expose an agent over HTTP in WSO2 Integrator is to attach a service to `ai:Listener`. The listener knows about `ai:ChatReqMessage` / `ai:ChatRespMessage` and manages per-session memory automatically.

```ballerina
import ballerina/ai;

service /support on new ai:Listener(8090) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        string response = check routerAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

You do not need to define your own chat request or response types -- `ai:ChatReqMessage` (`{ string sessionId; string message; }`) and `ai:ChatRespMessage` (`{ string message; }`) are provided by the module.

## What's Next

- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build your first agent
- [Adding Tools](/docs/genai/develop/agents/adding-tools) -- Define and register tools
- [Adding Memory](/docs/genai/develop/agents/adding-memory) -- Configure conversation persistence
- [AI Agent Observability](/docs/genai/develop/agents/agent-observability) -- Monitor agent behavior in production
- [AI Agent Evaluations](/docs/genai/develop/agents/agent-evaluations) -- Test and evaluate agent quality
