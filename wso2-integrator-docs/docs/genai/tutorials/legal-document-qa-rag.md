---
title: Building a Legal Document Q&A System with MCP and RAG
---

# Building a Legal Document Q&A System with MCP and RAG

This tutorial demonstrates how to build a question-and-answer system for legal documents using RAG for document retrieval and MCP for exposing the Q&A capability as a standardized tool. The system ingests legal documents (contracts, policies, compliance guides), stores them in a vector database, and answers questions grounded in the actual document content.

## What You'll Build

A two-part integration:

- **RAG ingestion and query service** -- Loads legal documents, chunks and embeds them, stores them in a vector database, and answers questions with source citations
- **MCP server** -- Exposes the Q&A capability as an MCP tool so other agents and applications can query your legal knowledge base programmatically

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- BI Copilot sign-in configured (for the default WSO2 model provider)
- Familiarity with [What is RAG?](../key-concepts/what-is-rag/index.md) and [What is MCP?](../key-concepts/what-is-mcp/index.md)

## Architecture Overview

```text
Legal Documents (.md, .txt, .pdf)
        │
        ▼
  ┌─────────────┐
  │  Ingestion   │  Load → Chunk → Embed → Store
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │ Vector Store │  In-memory (or Pinecone for production)
  └──────┬──────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌──────────┐
│  HTTP  │ │   MCP    │  Both expose the same query logic
│Service │ │  Server  │
└────────┘ └──────────┘
```

## Step 1: Create a New Project

1. Open VS Code and click the **WSO2 Integrator: BI** icon.
2. Click **Create New Integration**.
3. Name the project `LegalDocQA` and select a directory.

## Step 2: Ingest Legal Documents

Set up the RAG ingestion pipeline using the built-in `ai:TextDataLoader` and `ai:KnowledgeBase`:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

public function main() returns error? {
    // Ingest legal documents
    string[] documentPaths = [
        "./legal-docs/privacy-policy.md",
        "./legal-docs/terms-of-service.md",
        "./legal-docs/employee-handbook.md"
    ];

    foreach string path in documentPaths {
        ai:DataLoader loader = check new ai:TextDataLoader(path);
        ai:Document|ai:Document[] documents = check loader.load();
        check knowledgeBase.ingest(documents);
        io:println("Ingested: ", path);
    }

    io:println("All legal documents ingested successfully.");
}
```

Place your legal documents in a `legal-docs/` directory inside the project. The `TextDataLoader` supports `.md`, `.txt`, and `.pdf` files.

:::tip
For legal documents, the default chunking strategy works well for most cases. If you have very long contract clauses, consider increasing the chunk size to preserve clause boundaries. See [Chunking Documents](../build-ai-applications/rag/rag-ingestion/chunking-strategies.md) for configuration options.
:::

## Step 3: Build the Query Service

Create an HTTP service that retrieves relevant legal context and generates grounded answers:

```ballerina
import ballerina/http;

service /api on new http:Listener(8080) {
    resource function post query(@http:Payload QueryRequest request) returns QueryResponse|error {
        // Retrieve relevant chunks from the knowledge base
        ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(request.question, 10);
        ai:Chunk[] context = from ai:QueryMatch qm in queryMatches select qm.chunk;

        // Augment the query with retrieved context
        ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, request.question);

        // Generate a grounded response
        ai:ChatAssistantMessage response = check modelProvider->chat([
            {
                role: ai:SYSTEM,
                content: string `You are a legal document assistant. Answer questions using
                    ONLY the provided context from company legal documents.
                    Be precise and cite the specific document or section when possible.
                    If the context does not contain the answer, clearly state that the
                    information is not available in the current document set.
                    Do NOT provide legal advice -- only summarize what the documents say.`
            },
            augmentedQuery
        ]);

        return {
            answer: response.content.toString(),
            sourcesUsed: queryMatches.length()
        };
    }
}

type QueryRequest record {|
    string question;
|};

type QueryResponse record {|
    string answer;
    int sourcesUsed;
|};
```

:::warning
This system summarizes document content -- it does not provide legal advice. Always include a disclaimer in production deployments.
:::

## Step 4: Expose as an MCP Server

Expose the Q&A capability as an MCP server so other agents can query your legal knowledge base:

```ballerina
import ballerina/mcp;

listener mcp:Listener mcpListener = new (9090);

service mcp:Service /mcp on mcpListener {
    # Query the legal document knowledge base.
    # + question - The legal question to answer based on ingested documents
    # + return - Answer grounded in the legal documents with source count
    remote function queryLegalDocs(string question) returns QueryResponse|error {
        ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(question, 10);
        ai:Chunk[] context = from ai:QueryMatch qm in queryMatches select qm.chunk;

        ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, question);

        ai:ChatAssistantMessage response = check modelProvider->chat([
            {
                role: ai:SYSTEM,
                content: string `You are a legal document assistant. Answer questions using
                    ONLY the provided context from company legal documents.
                    Be precise and cite the specific document or section when possible.
                    If the context does not contain the answer, clearly state that.
                    Do NOT provide legal advice -- only summarize what the documents say.`
            },
            augmentedQuery
        ]);

        return {
            answer: response.content.toString(),
            sourcesUsed: queryMatches.length()
        };
    }
}
```

The MCP server automatically exposes `queryLegalDocs` as an MCP tool. Any MCP-compatible agent can discover and call it.

## Step 5: Connect an Agent to the MCP Server

Build a conversational agent that uses your legal Q&A MCP server as a tool:

```ballerina
final ai:McpToolKit legalMcpConn = check new ("http://localhost:9090/mcp");

final ai:Agent legalAssistant = check new (
    systemPrompt = {
        role: "Legal Document Assistant",
        instructions: string `You are a legal document assistant that helps users
            find information in company legal documents. Use the queryLegalDocs
            tool to search for answers. Always cite which document the answer
            comes from. Remind users that you summarize documents and do not
            provide legal advice.`
    },
    tools = [legalMcpConn],
    model = check ai:getDefaultModelProvider()
);
```

## Step 6: Test the System

**Start the services:**

```bash
# Terminal 1: Run the integration (starts both HTTP and MCP servers)
bal run
```

**Test the HTTP service:**

```bash
curl -X POST http://localhost:8080/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the data retention policy for customer records?"}'
```

**Test the conversational agent:**

```text
You: What does our privacy policy say about third-party data sharing?
Agent: According to the privacy policy document, third-party data sharing is permitted
       only with explicit user consent and for the purposes outlined in section 4.2...
       [Source: privacy-policy.md]

You: Are there any exceptions to that rule?
Agent: Yes, the privacy policy lists three exceptions in section 4.3:
       1. Legal obligations (court orders, regulatory requirements)
       2. Service providers under data processing agreements
       3. Anonymous, aggregated data that cannot identify individuals
       [Source: privacy-policy.md]
```

## Extend It

- **Add PDF support** -- The `ai:TextDataLoader` supports PDF files. Add contract PDFs to your `legal-docs/` directory.
- **Switch to a production vector store** -- Replace `ai:InMemoryVectorStore` with `pinecone:VectorStore` for persistence across restarts. See [Connecting to Vector Databases](../build-ai-applications/rag/rag-ingestion/connecting-to-vector-database.md).
- **Add metadata filtering** -- Tag documents by type (contract, policy, handbook) and filter retrieval results by category.
- **Add authentication** -- Secure the MCP server with bearer token authentication. See [Building AI Agents with MCP Servers](../build-ai-applications/mcp-integration/building-agents-with-mcp-servers.md).

## What's Next

- [RAG Querying](../build-ai-applications/rag/rag-query/rag-querying.md) -- Advanced retrieval configuration and query augmentation
- [Creating an MCP Server](../build-ai-applications/mcp-integration/exposing-services-as-mcp-server.md) -- Full MCP server reference
- [Building an HR Knowledge Base Agent with RAG](hr-knowledge-base-agent-rag.md) -- Another RAG tutorial with a different domain
