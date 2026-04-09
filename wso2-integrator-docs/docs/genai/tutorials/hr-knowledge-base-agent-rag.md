---
title: Building an HR Knowledge Base Agent with RAG
---

# Building an HR Knowledge Base Agent with RAG

This example demonstrates how to build a complete RAG-powered knowledge base that ingests documents, stores embeddings in a vector database, and serves context-aware answers through both an HTTP service and a conversational chat interface. It covers production-ready features like metadata filtering, source attribution, and a configurable chunking strategy.

## What You'll Build

A three-part integration:

- **Ingestion automation** -- Loads documents from a directory, chunks them with configurable strategies, embeds them, and stores them in a vector database
- **Query HTTP service** -- Accepts questions via REST, retrieves relevant context, and returns grounded answers with source citations
- **Chat interface** -- A conversational agent that wraps the query service for multi-turn Q&A

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An OpenAI API key (for embeddings and chat completions)
- Familiarity with [What is RAG?](../key-concepts/what-is-rag/index.md)

:::info
This example uses an in-memory vector store. For production, swap in Pinecone, Weaviate, or pgvector -- see [Connecting to Vector Databases](../build-ai-applications/rag/rag-ingestion/connecting-to-vector-database.md) for setup details.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `hr-knowledge-base` and choose a directory.

## Step 2: Set Up the Document Loader

Create a utility module that reads files from a directory and extracts metadata. This module supports markdown, plain text, and JSON files.

```ballerina
import ballerina/file;
import ballerina/io;

type DocumentRecord record {|
    string content;
    string filePath;
    string fileName;
    string fileType;
    int fileSize;
|};

function loadDocuments(string directory) returns DocumentRecord[]|error {
    file:MetaData[] entries = check file:readDir(directory);
    DocumentRecord[] documents = [];

    foreach file:MetaData entry in entries {
        string path = entry.absPath;
        if path.endsWith(".md") || path.endsWith(".txt") || path.endsWith(".json") {
            string content = check io:fileReadString(path);
            documents.push({
                content: content,
                filePath: path,
                fileName: check file:basename(path),
                fileType: path.endsWith(".md") ? "markdown" : (path.endsWith(".json") ? "json" : "text"),
                fileSize: content.length()
            });
        }
    }
    return documents;
}
```

## Step 3: Configure Chunking

Set up a configurable chunking strategy. Different document types benefit from different chunk sizes and overlap settings.

```ballerina
import ballerinax/ai.rag;

configurable int chunkSize = 512;
configurable int chunkOverlap = 50;

function chunkDocument(DocumentRecord doc) returns rag:Chunk[]|error {
    // Use larger chunks for markdown (better context preservation)
    int effectiveChunkSize = doc.fileType == "markdown" ? chunkSize * 2 : chunkSize;

    rag:Chunk[] chunks = check rag:chunkText(doc.content, {
        chunkSize: effectiveChunkSize,
        chunkOverlap: chunkOverlap
    });

    // Attach source metadata to each chunk
    foreach int i in 0 ..< chunks.length() {
        chunks[i].metadata = {
            source: doc.filePath,
            fileName: doc.fileName,
            fileType: doc.fileType,
            chunkIndex: i.toString()
        };
    }

    return chunks;
}
```

:::tip
For technical documentation, larger chunk sizes (800-1024 tokens) preserve more context. For FAQs, smaller chunks (256-512 tokens) give more precise retrieval. See [Chunking Documents](../build-ai-applications/rag/rag-ingestion/chunking-strategies.md) for guidance.
:::

## Step 4: Build the Ingestion Automation

Wire together the loader, chunker, and vector store into a single automation:

```ballerina
import ballerinax/openai.embeddings as embed;

configurable string openaiApiKey = ?;
configurable string docsDirectory = "./knowledge-base";

final embed:Client embeddingClient = check new ({auth: {token: openaiApiKey}});
final rag:InMemoryVectorStore vectorStore = new;

public function main() returns error? {
    DocumentRecord[] docs = check loadDocuments(docsDirectory);
    int totalChunks = 0;

    foreach DocumentRecord doc in docs {
        rag:Chunk[] chunks = check chunkDocument(doc);

        foreach rag:Chunk chunk in chunks {
            float[] embedding = check embeddingClient->/embeddings.post({
                model: "text-embedding-3-small",
                input: chunk.text
            }).data[0].embedding;

            check vectorStore.add({
                id: chunk.id,
                text: chunk.text,
                embedding: embedding,
                metadata: chunk.metadata
            });
            totalChunks += 1;
        }
    }

    io:println(string `Ingestion complete: ${docs.length()} documents, ${totalChunks} chunks indexed.`);
}
```

## Step 5: Create the Query HTTP Service

Build an HTTP service that retrieves relevant context and generates grounded answers with source attribution:

```ballerina
import ballerina/http;
import ballerinax/openai.chat as openai;

final openai:Client chatClient = check new ({auth: {token: openaiApiKey}});

type QueryRequest record {|
    string question;
    int topK = 3;
|};

type SourceReference record {|
    string fileName;
    string snippet;
|};

type QueryResponse record {|
    string answer;
    SourceReference[] sources;
|};

service /api on new http:Listener(8080) {

    resource function post query(@http:Payload QueryRequest request) returns QueryResponse|error {
        float[] queryEmbedding = check embeddingClient->/embeddings.post({
            model: "text-embedding-3-small",
            input: request.question
        }).data[0].embedding;

        rag:SearchResult[] results = check vectorStore.search(queryEmbedding, topK = request.topK);

        string context = results.'map(r => r.text).reduce(
            isolated function(string acc, string val) returns string => acc + "\n---\n" + val, ""
        );

        openai:ChatCompletion completion = check chatClient->/chat/completions.post({
            model: "gpt-4o",
            messages: [
                {role: "system", content: string `You are a knowledge base assistant. Answer questions using ONLY the provided context. If the context does not contain the answer, say "I don't have information about that in our knowledge base." Always be specific and cite which document your answer comes from.

                Context:
                ${context}`},
                {role: "user", content: request.question}
            ]
        });

        SourceReference[] sources = results.'map(r => ({
            fileName: r.metadata["fileName"].toString(),
            snippet: r.text.substring(0, 100) + "..."
        }));

        return {
            answer: completion.choices[0].message.content ?: "No answer generated.",
            sources: sources
        };
    }
}
```

## Step 6: Add a Chat Agent Wrapper

Layer a conversational agent on top of the query service so users can ask follow-up questions:

```ballerina
import ballerinax/ai.agent;

@agent:Tool {
    description: "Search the knowledge base to find answers to questions about company products, policies, and documentation."
}
isolated function queryKnowledgeBase(string question) returns string|error {
    float[] qEmbedding = check embeddingClient->/embeddings.post({
        model: "text-embedding-3-small",
        input: question
    }).data[0].embedding;

    rag:SearchResult[] results = check vectorStore.search(qEmbedding, topK = 3);

    return results.'map(r => string `[${r.metadata["fileName"].toString()}]: ${r.text}`)
        .reduce(isolated function(string acc, string val) returns string => acc + "\n\n" + val, "");
}

final agent:ChatAgent knowledgeAgent = check new (
    systemPrompt = string `You are a knowledge base assistant. Use the queryKnowledgeBase tool to answer user questions. Always cite the source document. If the knowledge base does not have an answer, say so clearly.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 20),
    tools = [queryKnowledgeBase]
);
```

## Step 7: Configure and Run

Add your settings to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
docsDirectory = "./knowledge-base"
chunkSize = 512
chunkOverlap = 50
```

:::warning
Never commit API keys to version control. Use environment variables or a secrets manager in production. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption).
:::

## Test It

**Test the HTTP service:**

```bash
curl -X POST http://localhost:8080/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the return policy for electronics?", "topK": 3}'
```

**Test the chat interface:**

1. Click **Run** in VS Code.
2. Open the built-in chat interface.
3. Try a multi-turn conversation:

```
You: What payment methods do you accept?
Agent: Based on our documentation [payments-faq.md], we accept Visa, Mastercard, American Express, and PayPal...

You: Do you support Apple Pay?
Agent: I don't have information about Apple Pay in our knowledge base. You may want to check with our support team for the latest payment options.
```

:::tip
Use the **Try-It** tool in the visual designer to test your query service interactively without writing curl commands.
:::

## Extend It

- **Add PDF ingestion** -- Extend the document loader to parse PDFs with a text extraction library.
- **Switch to a production vector store** -- Replace the in-memory store with Pinecone or pgvector for persistence across restarts.
- **Add metadata filtering** -- Filter search results by document type, date, or category before sending to the LLM.

## Source Code

Find the complete working project on GitHub: [rag-knowledge-base example](https://github.com/wso2/integrator-examples/tree/main/genai/rag-knowledge-base)

## What's Next

- [Chunking Documents](../build-ai-applications/rag/rag-ingestion/chunking-strategies.md) -- Advanced loading, parsing, and chunking strategies
- [RAG Querying](../build-ai-applications/rag/rag-query/rag-querying.md) -- Production patterns for RAG query services
- [AI Governance and Security](../reference/ai-governance-and-security.md) -- Guardrails and cost management
