---
sidebar_position: 5
title: "Build a RAG Query Service"
description: "Create an HTTP service that answers questions using retrieval-augmented generation from your knowledge base."
---

# Build a RAG Query Service

Once you have ingested documents into a vector store, you need a service that accepts user questions, retrieves relevant context, and returns LLM-generated answers grounded in your data. This guide walks you through building that query service as an HTTP endpoint in WSO2 Integrator.

## What You Will Build

An HTTP service with a `POST /query` endpoint that:

1. Accepts a question from the client
2. Embeds the question into a vector
3. Searches the vector store for relevant chunks
4. Augments the question with retrieved context
5. Sends the augmented prompt to an LLM
6. Returns the generated response to the client

## Step 1: Create the Project

1. Open VS Code and launch the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `rag_query_service`.
4. Choose **Service** as the project type.

## Step 2: Define Request and Response Types

```ballerina
import ballerina/http;
import ballerina/log;
import ballerinax/ai;

// Request payload
type QueryRequest record {|
    string question;
    int topK = 5;
|};

// Response payload
type QueryResponse record {|
    string answer;
    Source[] sources;
|};

type Source record {|
    string content;
    map<json> metadata;
    float score;
|};
```

## Step 3: Configure Connections

Set up the embedding provider, vector store, and LLM connections:

```ballerina
configurable string openaiApiKey = ?;
configurable string pineconeApiKey = ?;
configurable string pineconeEnv = ?;
configurable string pineconeIndex = ?;

// Embedding provider — must match the model used during ingestion
final ai:EmbeddingProvider embedder = check new ai:OpenAIEmbedder({
    apiKey: openaiApiKey,
    model: "text-embedding-3-small"
});

// Vector store connection
final ai:VectorStore vectorStore = check new ai:PineconeStore({
    apiKey: pineconeApiKey,
    environment: pineconeEnv,
    index: pineconeIndex
});

// LLM for answer generation
final ai:ChatClient llm = check new ai:OpenAIChatClient({
    apiKey: openaiApiKey,
    model: "gpt-4o"
});
```

:::info
For connection configuration details including API key management and authentication, see the [Connectors](/docs/connectors/configuration) section.
:::

## Step 4: Build the Service

Create the HTTP service with the query endpoint:

```ballerina
service /api on new http:Listener(9090) {

    resource function post query(QueryRequest request) returns QueryResponse|error {
        log:printInfo("Received query", question = request.question);

        // Step 1: Embed the question
        float[] queryVector = check embedder->embed(request.question);

        // Step 2: Retrieve relevant chunks from the vector store
        ai:SearchResult[] results = check vectorStore->search(
            queryVector, k = request.topK
        );

        if results.length() == 0 {
            return {
                answer: "I could not find any relevant information in the knowledge base.",
                sources: []
            };
        }

        // Step 3: Build the augmented prompt
        string context = buildContext(results);
        string augmentedPrompt = buildPrompt(request.question, context);

        // Step 4: Generate the answer
        string answer = check llm->chat(augmentedPrompt);

        // Step 5: Return the response with sources
        Source[] sources = results.map(r => <Source>{
            content: r.content,
            metadata: r.metadata,
            score: r.score
        });

        return {answer, sources};
    }
}
```

## Step 5: Implement Helper Functions

### Build Context from Retrieved Chunks

```ballerina
function buildContext(ai:SearchResult[] results) returns string {
    string context = "";
    foreach int i in 0 ..< results.length() {
        context += string `[Source ${i + 1}]: ${results[i].content}` + "\n\n";
    }
    return context;
}
```

### Build the Augmented Prompt

```ballerina
function buildPrompt(string question, string context) returns string {
    return string `You are a helpful assistant that answers questions based on the provided context.
Use only the information from the context below to answer the question.
If the context does not contain enough information, say so clearly.
Always cite which source(s) you used.

Context:
${context}

Question: ${question}

Answer:`;
}
```

:::tip
Keep your system prompt specific and instructive. Tell the LLM to stay within the provided context and to admit when it does not know — this reduces hallucination and builds user trust.
:::

## Step 6: Test the Service

Run the service and send a test query:

```bash
# Start the service
bal run

# Send a query
curl -X POST http://localhost:9090/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I configure authentication?", "topK": 5}'
```

Expected response:

```json
{
  "answer": "To configure authentication, you need to...",
  "sources": [
    {
      "content": "Authentication can be configured by...",
      "metadata": {"source": "auth-guide.md", "section": "setup"},
      "score": 0.92
    }
  ]
}
```

## Adding a System Prompt for Domain Context

For domain-specific applications, add a system message that primes the LLM with your domain knowledge:

```ballerina
function buildPrompt(string question, string context) returns string {
    return string `You are a technical support assistant for Acme Corp products.
You answer questions using only the documentation provided in the context.
Respond in a professional, concise tone.
If the documentation does not cover the topic, direct the user to support@acme.com.

Context:
${context}

Question: ${question}

Answer:`;
}
```

## Error Handling

Handle common failure scenarios gracefully:

```ballerina
resource function post query(QueryRequest request) returns QueryResponse|http:InternalServerError {
    do {
        // ... main logic ...
        return {answer, sources};
    } on fail error e {
        log:printError("Query failed", 'error = e);
        return <http:InternalServerError>{
            body: {message: "Failed to process your query. Please try again."}
        };
    }
}
```

:::warning
Never expose raw error details to clients. Log the full error server-side and return a user-friendly message. For production error handling patterns, see [Error Handling](/docs/develop/build/error-handling).
:::

For production deployment of your RAG service, including scaling and high availability, see [Deploy & Operate](/docs/deploy-operate).

## What's Next

- [Choose Chunking & Embedding Strategies](chunking-embedding.md) — Optimize retrieval quality by tuning your chunking approach
- [RAG Architecture Overview](architecture-overview.md) — Review the full RAG pattern and design decisions
- [AI-Powered Customer Support Agent](/docs/genai/tutorials/customer-support-agent) — Build a full agent that uses RAG for FAQ retrieval
