---
sidebar_position: 3
title: "Expose an Agent as an API"
description: Expose agents as HTTP/REST endpoints for programmatic access, and embed inline agents inside GraphQL resolvers and service logic.
---

# Expose an Agent as an API

While chat agents provide a built-in conversational interface, you often need agents that operate programmatically — called from other services, embedded in GraphQL resolvers, or triggered by webhooks. WSO2 Integrator supports two patterns for this: HTTP-exposed agents and inline agents.

This guide covers both approaches so you can choose the right one for your use case.

## HTTP-Exposed Chat Agent

A chat agent automatically exposes a REST endpoint. You can call it from any HTTP client without using the built-in chat interface:

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

final agent:ChatAgent supportAgent = check new (
    systemPrompt = "You are a customer support agent for an e-commerce platform.",
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [lookupOrder, checkInventory, createTicket]
);
```

### Request and Response Format

Send a POST request to the agent's chat endpoint:

```bash
curl -X POST http://localhost:9090/supportAgent/chat \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "user-123",
    "message": "Where is my order #45678?"
  }'
```

Response:

```json
{
  "sessionId": "user-123",
  "response": "Your order #45678 is currently in transit. It was shipped on March 8 and the estimated delivery date is March 12.",
  "toolCalls": [
    {
      "tool": "lookupOrder",
      "arguments": {"orderId": "45678"},
      "result": "shipped"
    }
  ]
}
```

### Session Management

The `sessionId` field links messages to a conversation session. Messages with the same `sessionId` share memory context:

- **Same session** — The agent remembers previous messages in the conversation.
- **New session** — A fresh conversation with no prior context.
- **No session ID** — Each message is treated as an independent interaction.

:::tip
For production deployments, use a session store (Redis, database) to persist sessions across restarts. See [Configure Agent Memory](/docs/genai/agents/memory-configuration) for persistent memory options.
:::

### Authentication

Protect your agent endpoint with standard HTTP authentication:

```ballerina
import ballerina/http;

listener http:Listener securedListener = check new (9090, {
    secureSocket: {
        key: {certFile: "cert.pem", keyFile: "key.pem"}
    }
});

@http:ServiceConfig {
    auth: [
        {oauth2IntrospectionConfig: {url: "https://auth.example.com/introspect"}}
    ]
}
service on securedListener {
    // Agent resources are now protected
}
```

See [Authentication](/docs/deploy-operate/secure/authentication) for full details on securing service endpoints.

## Inline Agents

Inline agents run inside your existing service logic rather than exposing their own endpoint. You create them on demand and call them programmatically.

### Inline Agent in an HTTP Service

```ballerina
import ballerina/http;
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

service /api on new http:Listener(8080) {

    resource function post summarize(@http:Payload record {|string document;|} request) returns string|error {
        agent:InlineAgent summarizer = check new (
            systemPrompt = "Summarize the given document in 3 bullet points.",
            model = check new openai:Client({auth: {token: openaiApiKey}})
        );
        agent:ChatMessage response = check summarizer.run(request.document);
        return response.content;
    }
}
```

### Inline Agent in a GraphQL Resolver

Inline agents work naturally inside GraphQL resolvers. This pattern lets you add AI capabilities to specific fields without restructuring your entire service:

```ballerina
import ballerina/graphql;
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

@agent:Tool
isolated function searchProducts(string query) returns record{}[]|error {
    // Search product database
    return [{name: "Widget Pro", price: 29.99, rating: 4.5}];
}

@agent:Tool
isolated function getProductReviews(string productId) returns string[]|error {
    return ["Great product!", "Works as expected."];
}

service /graphql on new graphql:Listener(4000) {

    resource function get recommendation(string userQuery) returns string|error {
        agent:InlineAgent recommender = check new (
            systemPrompt = string `You recommend products based on user preferences.
                Always explain why you recommend each product.`,
            model = check new openai:Client({auth: {token: openaiApiKey}}),
            tools = [searchProducts, getProductReviews]
        );
        agent:ChatMessage response = check recommender.run(userQuery);
        return response.content;
    }

    remote function addReview(string productId, string reviewText) returns string|error {
        agent:InlineAgent moderator = check new (
            systemPrompt = "You moderate product reviews. Flag inappropriate content.",
            model = check new openai:Client({auth: {token: openaiApiKey}})
        );
        agent:ChatMessage result = check moderator.run(reviewText);
        return result.content;
    }
}
```

:::info
Inline agents are stateless by default — they do not maintain memory across calls. If you need conversational context, pass the history explicitly or use a chat agent instead.
:::

## Choosing Between Chat Agents and Inline Agents

| Factor | Chat Agent | Inline Agent |
|---|---|---|
| User-facing conversations | Best fit | Not ideal |
| Programmatic single-task calls | Possible | Best fit |
| Built-in memory management | Yes | Manual |
| Built-in chat interface | Yes | No |
| Embedded in existing services | No | Yes |
| GraphQL resolver integration | No | Yes |

## What's Next

- [Configure Agent Memory](/docs/genai/agents/memory-configuration) — Set up memory for chat agents and stateful inline agents
- [Bind Tools to Agents](/docs/genai/agents/tool-binding) — Give agents the ability to call external services
- [Orchestrate Multiple Agents](/docs/genai/agents/multi-agent-orchestration) — Route between specialized agents
