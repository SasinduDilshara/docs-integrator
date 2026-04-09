---
title: "RAG Querying"
description: "Retrieve relevant context from your vector knowledge base, augment queries, and generate grounded LLM responses."
---

# RAG Querying

Once you have ingested documents into a vector store, the next step is retrieving relevant context at query time and using it to generate grounded LLM responses. This guide covers the complete query pipeline: retrieval, augmentation, and response generation.

## Knowledge Base Retrieval

WSO2 Integrator provides a `knowledgeBase.retrieve()` method that handles embedding the query, searching the vector store, and returning matched chunks -- all in a single call.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

public function main() returns error? {
    // Ingestion
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();
    check knowledgeBase.ingest(documents);
    io:println("Ingestion successful");

    // Query
    string query = "How many annual leave days can a full-time employee carry forward to the next year?";
    ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(query, 10);
    ai:Chunk[] context = from ai:QueryMatch queryMatch in queryMatches select queryMatch.chunk;
    ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, query);
    ai:ChatAssistantMessage assistantMessage = check modelProvider->chat(augmentedQuery);
    io:println("\nQuery: ", query);
    io:println("Answer: ", assistantMessage.content);
}
```

## How Retrieval Works

When you call `knowledgeBase.retrieve(query, topK)`, the following happens behind the scenes:

1. The query string is converted into a vector embedding using the configured embedding provider.
2. The vector store performs a similarity search, comparing the query vector against all stored chunk vectors.
3. The top-K most similar chunks are returned as `ai:QueryMatch[]` records.

Each `QueryMatch` contains the matched chunk along with its similarity score, giving you both the content and a measure of relevance.

## Configuring Top-K

The second parameter to `retrieve()` controls how many chunks are returned. The right value depends on your use case:

| Top-K | Use Case | Trade-off |
|-------|----------|-----------|
| **3-5** | Focused, single-topic questions | Lower token cost, risk of missing context |
| **5-10** | General-purpose queries (recommended) | Balanced coverage and cost |
| **10-20** | Complex, multi-part questions | Broad context, higher token cost |

```ballerina
// Retrieve the top 10 most relevant chunks
ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(query, 10);
```

:::tip
Start with a top-K of 10 and adjust based on answer quality. If answers are missing key details, increase it. If answers include irrelevant information, decrease it.
:::

## Extracting Chunks from Query Matches

The `retrieve()` method returns `ai:QueryMatch[]` records. To pass the content to the LLM, extract the chunks using a query expression:

```ballerina
ai:Chunk[] context = from ai:QueryMatch queryMatch in queryMatches select queryMatch.chunk;
```

This gives you an array of chunks that you can use for query augmentation in the next step.

## Source Attribution

When your chunks carry metadata (set during ingestion), you can use that metadata to provide source attribution in your responses. This tells users exactly which documents informed the answer.

```ballerina
// Extract source information from retrieved chunks
ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(query, 5);

string[] sources = [];
foreach ai:QueryMatch match in queryMatches {
    ai:Chunk chunk = match.chunk;
    if chunk.metadata is map<anydata> {
        map<anydata> meta = <map<anydata>>chunk.metadata;
        string? source = meta["source"] is string ? <string>meta["source"] : ();
        if source is string {
            sources.push(source);
        }
    }
}
```

## Query Augmentation

WSO2 Integrator provides a built-in `ai:augmentUserQuery()` function that takes retrieved chunks and the original question, then assembles them into a structured prompt the LLM can use.

```ballerina
ai:Chunk[] context = from ai:QueryMatch queryMatch in queryMatches select queryMatch.chunk;
ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, query);
```

### How Augmentation Works

The `ai:augmentUserQuery()` function:

1. Takes the array of retrieved `ai:Chunk[]` records and the original query string.
2. Assembles them into a `ai:ChatUserMessage` that includes both the context and the question.
3. Returns a message ready to send to the model provider.

## Model Provider Connection

WSO2 Integrator supports multiple model providers. The default provider requires no API key management:

```ballerina
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();
```

Send the augmented query to generate a grounded response:

```ballerina
ai:ChatAssistantMessage assistantMessage = check modelProvider->chat(augmentedQuery);
io:println("Answer: ", assistantMessage.content);
```

:::tip Model flexibility
The default WSO2 model provider gives you instant access without managing API keys. When you are ready for production, you can swap it for **OpenAI**, **Anthropic**, **Azure OpenAI**, **Google PaLM**, or a **local model** -- the rest of the service logic stays exactly the same.
:::

## Building a Complete RAG Query Service

You can expose the RAG pipeline as an HTTP service. The following shows the complete flow using the built-in components:

### Visual Designer Approach

1. Create an HTTP service with a POST resource (e.g., `/query`).
2. Add a **retrieve** action from your knowledge base to fetch relevant chunks.
3. Add an **Augment Query** step to combine context with the user query.
4. Add a **Model Provider** to generate the response.
5. Return the generated response.

The visual designer wires these steps together in a drag-and-drop flow.

### Pro-Code Approach

```ballerina
import ballerina/http;
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

service / on new http:Listener(9090) {
    resource function post query(@http:Payload string userQuery) returns string|error {
        ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(userQuery, 10);
        ai:Chunk[] context = from ai:QueryMatch queryMatch in queryMatches select queryMatch.chunk;
        ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, userQuery);
        ai:ChatAssistantMessage assistantMessage = check modelProvider->chat(augmentedQuery);
        return assistantMessage.content.toString();
    }
}
```

Test the service:

```bash
curl -X POST http://localhost:9090/query \
  -H "Content-Type: application/json" \
  -d '"Who should I contact for refund approval?"'
```

:::warning Response may vary
Since this integration involves an LLM call, response values may not always be identical across different executions. The LLM generates answers based on the retrieved context, but phrasing and detail level can differ each time.
:::

## Custom RAG with External Services

For production deployments, you can build a custom pipeline using external services like Pinecone for vector storage and Azure OpenAI for embeddings and text generation. This gives you persistent vector storage and fine-grained control over each pipeline stage.

The custom pipeline consists of four discrete functions:

1. **getEmbeddings** -- Convert the user query to a vector using Azure OpenAI embeddings.
2. **retrieveData** -- Search Pinecone for the most relevant document chunks.
3. **augment** -- Build a grounded prompt with the retrieved context.
4. **generateText** -- Call the LLM to produce the final answer.

## System Prompts for Domain Context

For domain-specific applications, add a system message that primes the LLM with your domain knowledge. This reduces hallucination and keeps answers within scope.

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

:::tip
Keep your augmentation prompt specific and instructive. Tell the LLM to stay within the provided context and to admit when it does not know -- this reduces hallucination and builds user trust.
:::

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

## Visual Designer Retrieval

You can also perform retrieval through the visual designer:

1. In the flow editor, hover over the flow line and click the **+** icon.
2. Select **Knowledge Bases** under the **AI** section.
3. Click on your knowledge base (e.g., `knowledgeBase`).
4. Click on **retrieve** to open the configuration panel.
5. Set the **Query** input to your user query variable.
6. Set the **Result** variable to store the matched chunks.
7. Click **Save**.

:::note
The retrieve action embeds the user query, searches the vector store for the most relevant chunks, and returns them as structured context -- all in a single step.
:::

## Comparing Built-in vs Custom Approaches

| Aspect | Built-in Components | Custom External Services |
|---|---|---|
| **Setup time** | Minutes | Hours |
| **Vector store** | In-memory (session-scoped) | Pinecone (persistent) |
| **Embedding provider** | Built-in | Azure OpenAI |
| **LLM provider** | WSO2 default (swappable) | Azure OpenAI |
| **Best for** | Prototyping, tutorials, demos | Production, enterprise deployments |
| **Data persistence** | Lost on restart | Persisted in Pinecone |

:::tip
Start with the built-in components to validate your RAG logic quickly, then migrate to custom external services when you need persistent storage, custom embedding models, or enterprise-grade infrastructure.
:::

## What's Next

- [Chunking Strategies](../rag-ingestion/chunking-strategies.md) -- Optimize how documents are split for better retrieval quality
- [Connecting to a Vector Database](../rag-ingestion/connecting-to-vector-database.md) -- Set up persistent vector stores for production
