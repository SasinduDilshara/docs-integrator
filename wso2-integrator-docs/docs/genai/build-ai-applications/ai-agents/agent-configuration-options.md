---
sidebar_position: 4
title: "Agent Configuration Options"
description: Multi-agent orchestration patterns, shared context, and advanced agent configuration for complex workflows.
---

# Agent Configuration Options

This page covers multi-agent orchestration patterns and advanced configuration options for building complex agent workflows in WSO2 Integrator.

## Multi-Agent Orchestration

A single agent works well for focused tasks, but complex workflows often require multiple specialized agents collaborating. WSO2 Integrator lets you create agent networks where a supervisor agent routes requests to specialists, agents hand off conversations, and context flows between them.

### When to Use Multi-Agent Orchestration

Use multiple agents when:

- A single system prompt cannot cover all required behaviors
- Different tasks need different tools or model configurations
- You want to isolate failures -- one agent's error should not crash the entire system
- Specialists need different LLM models (e.g., a fast model for classification, a powerful model for generation)

:::info
Start with a single agent and split into multiple agents only when complexity demands it. Over-engineering with multiple agents adds latency and debugging overhead.
:::

### Supervisor Pattern

The supervisor pattern uses a routing agent that analyzes the user's intent and delegates to the appropriate specialist:

```ballerina
import ballerina/ai;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

// --- Specialist Agents ---

final agent:ChatAgent billingAgent = check new (
    systemPrompt = string `You handle billing questions: invoices, payments, refunds.
        Be precise with amounts and dates.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    tools = [lookupInvoice, processRefund, getPaymentHistory]
);

final agent:ChatAgent technicalAgent = check new (
    systemPrompt = string `You handle technical support: troubleshooting, configuration, bug reports.
        Always ask for error messages and steps to reproduce.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    tools = [searchKnowledgeBase, checkServiceStatus, createBugReport]
);

final agent:ChatAgent shippingAgent = check new (
    systemPrompt = string `You handle shipping and delivery: tracking, address changes, delivery issues.
        Always confirm the order number first.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    tools = [trackShipment, updateAddress, reportDeliveryIssue]
);

// --- Supervisor Agent ---

@agent:Tool {description: "Route to the billing specialist for invoice, payment, or refund questions"}
isolated function routeToBilling(string userMessage) returns string|error {
    agent:ChatMessage response = check billingAgent.chat(userMessage);
    return response.content;
}

@agent:Tool {description: "Route to the technical support specialist for troubleshooting and bug reports"}
isolated function routeToTechnical(string userMessage) returns string|error {
    agent:ChatMessage response = check technicalAgent.chat(userMessage);
    return response.content;
}

@agent:Tool {description: "Route to the shipping specialist for delivery and tracking questions"}
isolated function routeToShipping(string userMessage) returns string|error {
    agent:ChatMessage response = check shippingAgent.chat(userMessage);
    return response.content;
}

final agent:ChatAgent supervisor = check new (
    systemPrompt = string `You are a customer support supervisor. Your job is to:
        1. Understand the user's intent.
        2. Route to the correct specialist (billing, technical, or shipping).
        3. If the intent is unclear, ask a clarifying question.
        Never try to answer domain-specific questions yourself.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [routeToBilling, routeToTechnical, routeToShipping]
);
```

### Direct Handoff Pattern

In the handoff pattern, agents transfer control to each other directly. The current agent decides when it cannot handle a request and passes the conversation to another agent:

```ballerina
import ballerina/ai;

// Sales agent that can hand off to support
@agent:Tool {description: "Hand off to the technical support agent when the user has a technical issue"}
isolated function handoffToSupport(string conversationSummary) returns string|error {
    agent:ChatMessage response = check technicalAgent.chat(
        string `[Handoff from Sales] Context: ${conversationSummary}`
    );
    return response.content;
}

final agent:ChatAgent salesAgent = check new (
    systemPrompt = string `You are a sales assistant. Help users choose products and pricing plans.
        If the user has a technical issue, hand off to the support agent with a summary of the conversation.`,
    model = myModel,
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [getProductInfo, getPricing, handoffToSupport]
);
```

:::tip
When handing off between agents, always pass a conversation summary. The receiving agent does not share memory with the sending agent, so context must be transferred explicitly.
:::

### Parallel Fan-Out Pattern

For tasks that require input from multiple specialists simultaneously, use a fan-out pattern. A coordinator agent calls multiple specialists in parallel and synthesizes their responses:

```ballerina
import ballerina/ai;

@agent:Tool {description: "Get a legal review of the proposed action"}
isolated function getLegalReview(string proposal) returns string|error {
    agent:InlineAgent legalAgent = check new (
        systemPrompt = "You review proposals for legal compliance. Be concise.",
        model = myModel
    );
    agent:ChatMessage response = check legalAgent.run(proposal);
    return response.content;
}

@agent:Tool {description: "Get a financial impact assessment of the proposed action"}
isolated function getFinancialReview(string proposal) returns string|error {
    agent:InlineAgent financeAgent = check new (
        systemPrompt = "You assess the financial impact of proposals. Include estimated costs.",
        model = myModel
    );
    agent:ChatMessage response = check financeAgent.run(proposal);
    return response.content;
}

@agent:Tool {description: "Get a technical feasibility assessment"}
isolated function getTechnicalReview(string proposal) returns string|error {
    agent:InlineAgent techAgent = check new (
        systemPrompt = "You assess technical feasibility. Identify risks and dependencies.",
        model = myModel
    );
    agent:ChatMessage response = check techAgent.run(proposal);
    return response.content;
}

final agent:ChatAgent coordinator = check new (
    systemPrompt = string `You coordinate proposal reviews. For each proposal:
        1. Send it to legal, financial, and technical reviewers.
        2. Synthesize their feedback into a unified recommendation.
        3. Highlight any conflicts between reviewers.`,
    model = myModel,
    tools = [getLegalReview, getFinancialReview, getTechnicalReview]
);
```

## Shared Context Between Agents

When agents need to share data beyond conversation summaries, use a shared context store:

```ballerina
import ballerina/ai;

// Shared context accessible by all agents
final map<json> sharedContext = {};

@agent:Tool
isolated function setContextValue(string key, string value) returns string {
    lock {
        sharedContext[key] = value;
    }
    return string `Context updated: ${key} = ${value}`;
}

@agent:Tool
isolated function getContextValue(string key) returns string {
    lock {
        json? value = sharedContext[key];
        return value is () ? "Not found" : value.toString();
    }
}
```

Bind `setContextValue` and `getContextValue` as tools to any agent that needs to read or write shared state.

## Best Practices

- **Keep specialist agents focused.** Each agent should handle one domain with a clear system prompt.
- **Limit tool exposure.** Specialists should only have tools relevant to their domain.
- **Use inline agents for stateless specialists.** If a specialist does not need memory, use an inline agent for lower overhead.
- **Log handoffs and routing decisions.** Multi-agent interactions are harder to debug -- tracing is essential.
- **Set iteration limits.** Prevent runaway agent loops by configuring maximum iterations on each agent.
- **Start simple.** Begin with a single agent and split into multiple agents only when complexity demands it.
- **Transfer context explicitly.** When handing off between agents, pass a conversation summary since agents do not share memory by default.

## What's Next

- [Creating an AI Agent](creating-an-ai-agent.md) -- Review the fundamentals of agent creation
- [Adding Tools to an Agent](adding-tools-to-an-agent.md) -- Define and configure the tools your specialist agents use
- [Adding Memory to an Agent](adding-memory-to-an-agent.md) -- Configure memory strategies for context management
