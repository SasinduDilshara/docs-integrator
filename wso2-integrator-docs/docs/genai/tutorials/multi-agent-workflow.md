---
sidebar_position: 3
title: "Orchestrate a Multi-Agent Workflow"
description: Build a supervisor agent that routes customer requests to specialized billing, shipping, and technical support agents in WSO2 Integrator.
---

# Orchestrate a Multi-Agent Workflow

**Time:** 30-45 minutes

You will build a multi-agent system where a supervisor agent analyzes incoming customer requests and routes them to the right specialist — billing, shipping, or technical support. Each specialist agent has its own tools, system prompt, and domain expertise. This pattern scales to any number of specialists and keeps each agent focused on what it does best.

## What You'll Build

A four-agent system:

- **Supervisor agent** — Classifies requests and routes to the correct specialist
- **Billing specialist** — Handles invoice lookups, payment status, and refund processing
- **Shipping specialist** — Tracks packages, updates delivery addresses, and handles shipping issues
- **Technical specialist** — Troubleshoots product issues and searches a technical knowledge base

## What You'll Learn

- How to build specialist agents with focused tool sets
- How to create a supervisor agent with routing logic
- How to configure handoff protocols between agents
- How to test routing scenarios across multiple specialists

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An OpenAI API key
- Familiarity with the [Build a Conversational Agent](/docs/genai/quick-starts/conversational-agent) quick start

:::info
This tutorial uses OpenAI. You can substitute any supported provider — see [AI & LLM Connectors](/docs/connectors/ai-llms) for connection configuration.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `multi-agent-support` and choose a directory.

## Step 2: Build the Billing Specialist Agent

Create the billing specialist with tools for invoice lookup and refund processing:

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;
import ballerina/http;

configurable string openaiApiKey = ?;

// --- Billing Specialist ---

@agent:Tool {
    description: "Look up an invoice by invoice ID or customer email. Returns invoice amount, status, and due date."
}
isolated function lookupInvoice(string invoiceId) returns string|error {
    http:Client billingApi = check new ("https://billing.example.com");
    json invoice = check billingApi->/api/invoices/[invoiceId];
    return invoice.toJsonString();
}

@agent:Tool {
    description: "Check payment status for a given invoice ID. Returns whether the payment is pending, completed, or failed."
}
isolated function checkPaymentStatus(string invoiceId) returns string|error {
    http:Client billingApi = check new ("https://billing.example.com");
    json status = check billingApi->/api/payments/[invoiceId]/status;
    return status.toJsonString();
}

@agent:Tool {
    description: "Initiate a refund for a given invoice ID. Requires a reason for the refund."
}
isolated function initiateRefund(string invoiceId, string reason) returns string|error {
    http:Client billingApi = check new ("https://billing.example.com");
    json response = check billingApi->/api/refunds.post({invoiceId, reason});
    return string `Refund initiated for invoice ${invoiceId}. Reference: ${check response.refundId}`;
}

final agent:ChatAgent billingAgent = check new (
    systemPrompt = string `You are a billing specialist. You help customers with invoice inquiries, payment status checks, and refund requests.
        - Always verify the invoice ID before processing any request.
        - For refunds, ask the customer for the reason before initiating.
        - Be precise with amounts and dates.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 15),
    tools = [lookupInvoice, checkPaymentStatus, initiateRefund]
);
```

## Step 3: Build the Shipping Specialist Agent

Create the shipping specialist with package tracking and address management tools:

```ballerina
@agent:Tool {
    description: "Track a package by order ID. Returns current location, status, and estimated delivery date."
}
isolated function trackPackage(string orderId) returns string|error {
    http:Client shippingApi = check new ("https://shipping.example.com");
    json tracking = check shippingApi->/api/tracking/[orderId];
    return tracking.toJsonString();
}

@agent:Tool {
    description: "Update the delivery address for an order. Only works if the package has not shipped yet."
}
isolated function updateDeliveryAddress(string orderId, string newAddress) returns string|error {
    http:Client shippingApi = check new ("https://shipping.example.com");
    json response = check shippingApi->/api/orders/[orderId]/address.put({address: newAddress});
    return string `Delivery address updated for order ${orderId}.`;
}

@agent:Tool {
    description: "Report a shipping issue such as a lost, damaged, or delayed package."
}
isolated function reportShippingIssue(string orderId, string issueType, string description) returns string|error {
    http:Client shippingApi = check new ("https://shipping.example.com");
    json response = check shippingApi->/api/issues.post({orderId, issueType, description});
    return string `Shipping issue reported. Case ID: ${check response.caseId}`;
}

final agent:ChatAgent shippingAgent = check new (
    systemPrompt = string `You are a shipping specialist. You help customers track packages, update delivery addresses, and resolve shipping issues.
        - Always confirm the order ID before taking action.
        - Explain any limitations (e.g., address changes only before shipment).
        - Show empathy for delayed or lost packages.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 15),
    tools = [trackPackage, updateDeliveryAddress, reportShippingIssue]
);
```

## Step 4: Build the Technical Specialist Agent

Create the technical specialist with troubleshooting and knowledge base tools:

```ballerina
import ballerinax/ai.rag;
import ballerinax/openai.embeddings as embed;

final embed:Client techEmbeddingClient = check new ({auth: {token: openaiApiKey}});
final rag:InMemoryVectorStore techKnowledgeBase = new;

@agent:Tool {
    description: "Search the technical knowledge base for solutions to product issues, setup guides, and troubleshooting steps."
}
isolated function searchTechDocs(string query) returns string|error {
    float[] queryEmbedding = check techEmbeddingClient->/embeddings.post({
        model: "text-embedding-3-small",
        input: query
    }).data[0].embedding;

    rag:SearchResult[] results = check techKnowledgeBase.search(queryEmbedding, topK = 3);
    return results.'map(r => r.text).reduce(
        isolated function(string acc, string val) returns string => acc + "\n---\n" + val, ""
    );
}

@agent:Tool {
    description: "Create a technical support ticket for issues that require engineering investigation."
}
isolated function createTechTicket(string subject, string description, string severity) returns string|error {
    http:Client supportApi = check new ("https://support.example.com");
    json response = check supportApi->/api/tickets.post({subject, description, severity, team: "engineering"});
    return string `Technical ticket created. ID: ${check response.ticketId}. Engineering will investigate within ${severity == "high" ? "4 hours" : "24 hours"}.`;
}

final agent:ChatAgent technicalAgent = check new (
    systemPrompt = string `You are a technical support specialist. You help customers troubleshoot product issues using the knowledge base and escalate to engineering when needed.
        - Search the knowledge base before escalating.
        - Walk customers through solutions step by step.
        - Only create tickets for issues that require engineering investigation.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 15),
    tools = [searchTechDocs, createTechTicket]
);
```

## Step 5: Create the Supervisor Agent

The supervisor classifies incoming requests and delegates to the appropriate specialist:

```ballerina
@agent:Tool {
    description: "Route a customer query to the billing specialist for invoice, payment, or refund questions."
}
isolated function routeToBilling(string customerMessage) returns string|error {
    agent:ChatResponse response = check billingAgent.chat(customerMessage);
    return response.message;
}

@agent:Tool {
    description: "Route a customer query to the shipping specialist for package tracking, delivery, or shipping issues."
}
isolated function routeToShipping(string customerMessage) returns string|error {
    agent:ChatResponse response = check shippingAgent.chat(customerMessage);
    return response.message;
}

@agent:Tool {
    description: "Route a customer query to the technical specialist for product issues, troubleshooting, or technical questions."
}
isolated function routeToTechnical(string customerMessage) returns string|error {
    agent:ChatResponse response = check technicalAgent.chat(customerMessage);
    return response.message;
}

final agent:ChatAgent supervisorAgent = check new (
    systemPrompt = string `You are a customer support supervisor. Your job is to understand the customer's request and route it to the right specialist.

        Routing rules:
        - Billing: invoices, payments, charges, refunds, pricing
        - Shipping: package tracking, delivery status, address changes, lost/damaged packages
        - Technical: product issues, bugs, setup help, how-to questions

        Guidelines:
        - Greet the customer and determine their need before routing.
        - If the request spans multiple areas, handle the primary concern first.
        - Always relay the specialist's response back to the customer naturally.
        - If unsure which specialist to route to, ask the customer to clarify.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [routeToBilling, routeToShipping, routeToTechnical]
);
```

:::tip
The supervisor pattern keeps each specialist focused and testable independently. You can add new specialists (e.g., returns, account management) without modifying existing agents.
:::

## Step 6: Configure and Run

Add your credentials to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
```

:::warning
Never commit API keys to version control. Use environment variables or a secrets manager in production. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption).
:::

## Test It

1. Click **Run** in VS Code.
2. Open the built-in chat interface.
3. Test routing across different specialists:

```
You: I haven't received my package for order ORD-5678
Agent: I understand you're concerned about your package. Let me check that for you.
  [Routes to shipping specialist]
  Your order ORD-5678 is currently in transit. It's at the regional distribution center
  and the estimated delivery is March 12, 2026. Would you like me to help with anything else?

You: Actually, I was also charged twice for this order
Agent: I'm sorry to hear about the double charge. Let me look into that for you.
  [Routes to billing specialist]
  I can see invoice INV-9012 associated with order ORD-5678. I see two charges of $49.99.
  Would you like me to initiate a refund for the duplicate charge?
```

:::tip
Open the **Trace** panel to see the full routing chain — from the supervisor's classification decision to the specialist's tool calls and responses.
:::

## Extend It

- **Add a returns specialist** — Create a new agent with return authorization and label generation tools.
- **Implement handoff memory** — Pass conversation context between agents so customers do not repeat themselves.
- **Add quality scoring** — Use an LLM-based evaluator to score agent responses and flag low-quality interactions.

## Source Code

Find the complete working project on GitHub: [multi-agent-workflow example](https://github.com/wso2/integrator-examples/tree/main/genai/multi-agent-workflow)

## What's Next

- [Multi-Agent Orchestration](/docs/genai/agents/multi-agent-orchestration) — Deep dive into supervisor and swarm patterns
- [Agent Tracing](/docs/genai/observability/agent-tracing) — End-to-end observability for multi-agent systems
- [Content Filtering](/docs/genai/guardrails/content-filtering) — Add safety guardrails across all agents
