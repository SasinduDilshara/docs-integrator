---
title: Building an IT Helpdesk Chatbot with Memory and Tools
---

# Building an IT Helpdesk Chatbot with Memory and Tools

This example demonstrates how to build an intelligent helpdesk agent that handles order inquiries, answers FAQs from a knowledge base using RAG, and creates support tickets when issues need escalation. It combines agent tool binding, retrieval-augmented generation, and external service integration into a single, production-ready integration.

## What You'll Build

A helpdesk agent with three capabilities:

- **Order lookup** -- Queries an order database to return status, tracking, and delivery estimates
- **FAQ retrieval** -- Searches a vector store of product documentation to answer common questions
- **Ticket creation** -- Creates support tickets in an external system when the agent cannot resolve an issue

The agent runs as an HTTP service with a built-in chat interface for testing.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An OpenAI API key (for chat completions and embeddings)
- Familiarity with the [Setting Up WSO2 Integrator](../getting-started/quick-start.md) guide

:::info
This example uses OpenAI as the model provider. You can substitute any supported provider -- see [AI & LLM Connectors](/docs/connectors/ai-llms) for connection configuration details.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `it-helpdesk-chatbot` and choose a directory.

## Step 2: Define the Order Lookup Tool

Create a tool that retrieves order information. In a real scenario, this connects to your order management system or database.

```ballerina
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;
import ballerinax/openai.embeddings as embed;
import ballerinax/ai.rag;
import ballerina/http;

configurable string openaiApiKey = ?;

type Order record {|
    string orderId;
    string status;
    string trackingNumber;
    string estimatedDelivery;
    string[] items;
|};

@agent:Tool {
    description: "Look up an order by its order ID. Returns order status, tracking information, and estimated delivery date."
}
isolated function lookupOrder(string orderId) returns string|error {
    // Connect to your order management system or database
    http:Client orderApi = check new ("https://orders.example.com");
    Order order = check orderApi->/api/orders/[orderId];

    return string `Order ${order.orderId}:
        Status: ${order.status}
        Tracking: ${order.trackingNumber}
        Estimated delivery: ${order.estimatedDelivery}
        Items: ${order.items.toString()}`;
}
```

:::tip
You can also use a [database connector](/docs/connectors/databases) to query orders directly from MySQL, PostgreSQL, or any supported database.
:::

## Step 3: Set Up RAG for FAQ Retrieval

Add a tool that searches your product FAQ knowledge base using vector similarity. First, set up the vector store and embedding client:

```ballerina
final embed:Client embeddingClient = check new ({auth: {token: openaiApiKey}});
final rag:InMemoryVectorStore faqStore = new;

@agent:Tool {
    description: "Search the product FAQ knowledge base for answers to common questions about returns, shipping, payments, and product features."
}
isolated function searchFAQ(string question) returns string|error {
    float[] queryEmbedding = check embeddingClient->/embeddings.post({
        model: "text-embedding-3-small",
        input: question
    }).data[0].embedding;

    rag:SearchResult[] results = check faqStore.search(queryEmbedding, topK = 3);

    if results.length() == 0 {
        return "No relevant FAQ entries found for this question.";
    }

    return results.'map(r => r.text).reduce(
        isolated function(string acc, string val) returns string => acc + "\n---\n" + val, ""
    );
}
```

:::info
This example uses an in-memory vector store for simplicity. For production deployments, connect to Pinecone, Weaviate, or pgvector. See [Connecting to Vector Databases](../build-ai-applications/rag/rag-ingestion/connecting-to-vector-database.md).
:::

## Step 4: Add the Ticket Creation Tool

When the agent cannot resolve a customer issue, it creates a support ticket:

```ballerina
type TicketRequest record {|
    string subject;
    string description;
    string priority; // "low", "medium", "high"
    string customerEmail;
|};

@agent:Tool {
    description: "Create a support ticket when the issue cannot be resolved directly. Use this for complex problems, refund requests, or complaints that need human review."
}
isolated function createTicket(string subject, string description, string priority, string customerEmail) returns string|error {
    http:Client ticketApi = check new ("https://support.example.com");
    TicketRequest ticket = {subject, description, priority, customerEmail};
    json response = check ticketApi->/api/tickets.post(ticket);

    string ticketId = check response.ticketId;
    return string `Ticket created successfully. Ticket ID: ${ticketId}. The support team will respond within 24 hours.`;
}
```

## Step 5: Configure the Agent with Memory

Wire all three tools together with a system prompt that guides the agent's behavior, and configure memory for multi-turn conversations:

```ballerina
final agent:ChatAgent supportAgent = check new (
    systemPrompt = string `You are a friendly and helpful customer support agent for Acme Corp.

        Guidelines:
        - Always greet the customer warmly and ask how you can help.
        - For order inquiries, ask for the order ID before looking it up.
        - For general questions, search the FAQ knowledge base first.
        - If you cannot resolve an issue, offer to create a support ticket.
        - Always ask for the customer's email before creating a ticket.
        - Never make up information — only use data from tools.
        - Be concise but empathetic in your responses.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [lookupOrder, searchFAQ, createTicket]
);
```

## Step 6: Ingest FAQ Documents

Create a `knowledge-base` directory in your project root and add your FAQ markdown files. Then add an ingestion function:

```ballerina
import ballerina/file;
import ballerina/io;

configurable string faqDirectory = "./knowledge-base";

function init() returns error? {
    string[] files = check file:readDir(faqDirectory)
        .filter(f => f.absPath.endsWith(".md"))
        .'map(f => f.absPath);

    foreach string filePath in files {
        string content = check io:fileReadString(filePath);
        rag:Chunk[] chunks = check rag:chunkText(content, {
            chunkSize: 512,
            chunkOverlap: 50
        });

        foreach rag:Chunk chunk in chunks {
            float[] chunkEmbedding = check embeddingClient->/embeddings.post({
                model: "text-embedding-3-small",
                input: chunk.text
            }).data[0].embedding;

            check faqStore.add({
                id: chunk.id,
                text: chunk.text,
                embedding: chunkEmbedding,
                metadata: {source: filePath}
            });
        }
    }
    io:println(string `Ingested ${files.length()} FAQ files.`);
}
```

## Step 7: Configure and Run

Add your credentials to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
faqDirectory = "./knowledge-base"
```

:::warning
Never commit API keys to version control. Use environment variables or a secrets manager for production deployments. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption).
:::

## Test It

1. Click **Run** in the VS Code toolbar.
2. Open the built-in chat interface.
3. Try a multi-turn conversation:

```
You: Hi, I need help with my order
Agent: Hello! I'd be happy to help you with your order. Could you share your order ID?

You: It's ORD-12345
Agent: I found your order. Here are the details:
  - Status: In Transit
  - Tracking: TRK-98765
  - Estimated delivery: March 15, 2026
  Is there anything else I can help you with?

You: What's your return policy?
Agent: Based on our FAQ, you can return any item within 30 days of delivery for a full refund...

You: I received a damaged item and want a refund
Agent: I'm sorry to hear that. I'll create a support ticket for your refund request. Could you share your email address?
```

:::tip
Open the **Trace** panel to inspect the full agent loop. You can see which tool the agent selected, the arguments it passed, and the response it received at each step.
:::

## Extend It

- **Add sentiment detection** -- Use an LLM-based guardrail to detect frustrated customers and auto-escalate to a human agent.
- **Connect to a real CRM** -- Replace the HTTP client stubs with a Salesforce or HubSpot connector.
- **Add conversation logging** -- Persist conversations for quality review using [AI Agent Observability](../build-ai-applications/ai-agents/ai-agent-observability.md).

## Source Code

Find the complete working project on GitHub: [customer-support-agent example](https://github.com/wso2/integrator-examples/tree/main/genai/customer-support-agent)

## What's Next

- [Creating an AI Agent](../build-ai-applications/ai-agents/creating-an-ai-agent.md) -- Understand the agent loop and reasoning process
- [AI Governance and Security](../reference/ai-governance-and-security.md) -- Add safety checks to validate agent inputs and outputs
- [Deploy to Production](/docs/deploy-operate/deploy/docker-kubernetes) -- Containerize and deploy your agent
