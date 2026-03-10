---
sidebar_position: 2
title: "Build a RAG Application"
description: Ingest documents into a vector store and build an HTTP query service in under 15 minutes.
---

# Build a RAG Application

**Time:** 10-15 minutes · **What you'll build:** A document ingestion pipeline and an HTTP query service that retrieves relevant context from a vector store and generates answers using an LLM.

You will set up an automation that chunks and embeds markdown files into a vector store, then create an HTTP service that answers questions grounded in your own data. This approach — Retrieval-Augmented Generation (RAG) — ensures the LLM only uses verified information from your knowledge base.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An OpenAI API key (for embeddings and chat completions)

:::info
This quick start uses an in-memory vector store for simplicity. For production, connect to Pinecone, Weaviate, or pgvector — see [Vector Database Connectivity](/docs/genai/rag/vector-database).
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `rag-knowledge-base` and choose a directory.

## Step 2: Set Up the Ingestion Automation

Create an automation that loads markdown files, splits them into chunks, and stores embeddings.

1. Click **+ Add Artifact** and select **Automation**.
2. Name it `DocumentIngestion`.

```ballerina
import ballerinax/ai.rag;
import ballerinax/openai.embeddings as embed;
import ballerina/file;
import ballerina/io;

configurable string openaiApiKey = ?;
configurable string docsDirectory = "./knowledge-base";

final embed:Client embeddingClient = check new ({auth: {token: openaiApiKey}});

final rag:InMemoryVectorStore vectorStore = new;

public function main() returns error? {
    string[] files = check file:readDir(docsDirectory)
        .filter(f => f.absPath.endsWith(".md"))
        .'map(f => f.absPath);

    foreach string filePath in files {
        string content = check io:fileReadString(filePath);

        rag:Chunk[] chunks = check rag:chunkText(content, {
            chunkSize: 512,
            chunkOverlap: 50
        });

        foreach rag:Chunk chunk in chunks {
            float[] embedding = check embeddingClient->/embeddings.post({
                model: "text-embedding-3-small",
                input: chunk.text
            }).data[0].embedding;

            check vectorStore.add({
                id: chunk.id,
                text: chunk.text,
                embedding: embedding,
                metadata: {source: filePath}
            });
        }
    }

    io:println(string `Ingested ${files.length()} files into vector store.`);
}
```

## Step 3: Create the Query Service

Add an HTTP service that accepts questions and returns LLM-generated answers grounded in retrieved context.

1. Click **+ Add Artifact** and select **Service**.
2. Name it `QueryService`.

```ballerina
import ballerina/http;
import ballerinax/ai.rag;
import ballerinax/openai.chat as openai;
import ballerinax/openai.embeddings as embed;

configurable string openaiApiKey = ?;

final openai:Client chatClient = check new ({auth: {token: openaiApiKey}});
final embed:Client embeddingClient = check new ({auth: {token: openaiApiKey}});

// Reference the same vector store populated by the ingestion automation
final rag:InMemoryVectorStore vectorStore = new;

type QueryRequest record {|
    string question;
|};

type QueryResponse record {|
    string answer;
    string[] sources;
|};

service /api on new http:Listener(8080) {

    resource function post query(@http:Payload QueryRequest request) returns QueryResponse|error {
        // Embed the user's question
        float[] queryEmbedding = check embeddingClient->/embeddings.post({
            model: "text-embedding-3-small",
            input: request.question
        }).data[0].embedding;

        // Retrieve the top 3 relevant chunks
        rag:SearchResult[] results = check vectorStore.search(queryEmbedding, topK = 3);

        // Build the context from retrieved chunks
        string context = results.'map(r => r.text).reduce(
            isolated function(string acc, string val) returns string => acc + "\n\n" + val, ""
        );

        // Generate a grounded answer
        openai:ChatCompletion completion = check chatClient->/chat/completions.post({
            model: "gpt-4o",
            messages: [
                {role: "system", content: string `Answer the question using ONLY the following context.
                    If the context doesn't contain the answer, say "I don't have enough information."
                    Context: ${context}`},
                {role: "user", content: request.question}
            ]
        });

        return {
            answer: completion.choices[0].message.content ?: "No answer generated.",
            sources: results.'map(r => r.metadata["source"].toString())
        };
    }
}
```

## Step 4: Add Your Documents

Create a `knowledge-base` directory in your project root and add a few markdown files:

```
knowledge-base/
  ├── product-overview.md
  ├── pricing-faq.md
  └── setup-guide.md
```

## Step 5: Configure and Run

Add your API key to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
docsDirectory = "./knowledge-base"
```

1. Run the ingestion automation first to populate the vector store.
2. Start the query service.
3. Test with a curl request:

```bash
curl -X POST http://localhost:8080/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I set up the product?"}'
```

:::tip
Use the **Try-It** tool in the visual designer to test your query service interactively without writing curl commands.
:::

## What's Next

- [Document Ingestion Pipelines](/docs/genai/rag/ingestion-pipelines) — Advanced loading, parsing, and chunking strategies
- [Chunking & Embedding Strategies](/docs/genai/rag/chunking-embedding) — Optimize retrieval quality with different approaches
- [Building a RAG Service](/docs/genai/rag/building-rag-service) — Production-ready patterns for RAG services
