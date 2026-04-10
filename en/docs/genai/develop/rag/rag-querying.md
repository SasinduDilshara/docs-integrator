---
sidebar_position: 4
title: RAG Querying
description: Build the complete RAG query pipeline with the WSO2 Integrator ai module -- retrieve relevant chunks and generate grounded answers.
---

# RAG Querying

The query pipeline is the runtime half of your RAG system. When a user asks a question, the pipeline embeds the query, searches for relevant chunks, assembles context, and generates a grounded answer.

In WSO2 Integrator, the `ai:KnowledgeBase` handles retrieval for you, and the `ai:ModelProvider` handles generation. This page covers two idiomatic query styles and a complete end-to-end example.

## Query Pipeline Overview

```
User Question --> knowledgeBase.retrieve --> Top-K Chunks --> modelProvider->chat --> Answer
```

## Complete RAG Example

The following program ingests a document, retrieves the most relevant chunks for a user question, and asks the LLM to answer using those chunks as context.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

public function main() returns error? {
    // 1. Load and ingest documents
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();
    check knowledgeBase.ingest(documents);
    io:println("Ingestion successful");

    // 2. Retrieve the most relevant chunks for the query
    string query = "How many annual leave days can a full-time employee carry forward?";
    ai:QueryMatch[] matches = check knowledgeBase.retrieve(query, 10);
    ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;

    // 3. Build an augmented user message and call the model
    ai:ChatUserMessage augmented = ai:augmentUserQuery(context, query);
    ai:ChatAssistantMessage answer = check modelProvider->chat(augmented);
    io:println("Answer: ", answer.content);
}
```

The `ai:augmentUserQuery` helper wraps the retrieved context and the user question into a single `ai:ChatUserMessage` that instructs the model to answer based only on the provided context. Passing that message to `modelProvider->chat` returns a grounded `ai:ChatAssistantMessage`.

## Two Query Styles

### Style A -- Augmented Chat Message

Use `ai:augmentUserQuery` when you want the framework to produce a standard retrieval-augmented prompt for you. This is the recommended style for most applications.

```ballerina
ai:QueryMatch[] matches = check knowledgeBase.retrieve(query, 10);
ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;

ai:ChatUserMessage augmented = ai:augmentUserQuery(context, query);
ai:ChatAssistantMessage response = check modelProvider->chat(augmented);

io:println(response.content);
```

### Style B -- Inline Context with `generate`

When you want full control over the prompt, retrieve the context and inline it into a `modelProvider->generate` call. This is handy for custom instruction formats or specialized output structures.

```ballerina
ai:QueryMatch[] matches = check knowledgeBase.retrieve(query, 10);
ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;

string answer = check modelProvider->generate(`Answer the query based on the
    following context:

    Context: ${context}

    Query: ${query}

    Base the answer only on the above context. If the answer is not contained
    within the context, respond with "I don't know".`);

io:println(answer);
```

Both styles use the same `ai:ModelProvider` and the same retrieved chunks, so you can mix and match them within a single application.

## Exposing RAG as an HTTP Service

Wrap the query pipeline in an HTTP service to make it available to other applications.

```ballerina
import ballerina/ai;
import ballerina/http;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

type QueryRequest record {|
    string question;
|};

type QueryResponse record {|
    string answer;
    int chunksUsed;
|};

service /rag on new http:Listener(8090) {

    resource function post query(@http:Payload QueryRequest request)
            returns QueryResponse|error {
        ai:QueryMatch[] matches = check knowledgeBase.retrieve(request.question, 5);
        ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;

        if context.length() == 0 {
            return {
                answer: "I couldn't find any relevant information to answer your question.",
                chunksUsed: 0
            };
        }

        ai:ChatUserMessage augmented = ai:augmentUserQuery(context, request.question);
        ai:ChatAssistantMessage response = check modelProvider->chat(augmented);

        return {
            answer: response.content ?: "",
            chunksUsed: context.length()
        };
    }
}
```

## Testing the Service

```bash
curl -X POST http://localhost:8090/rag/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the annual leave carry-forward policy?"}'
```

## Tuning Retrieval

The `topK` parameter on `retrieve` controls how many nearest chunks are returned. Start with `5`-`10` and adjust based on your corpus:

- **Small `topK` (1-3)** - tight, focused answers; faster calls; risk of missing context.
- **Medium `topK` (5-10)** - a good default for most use cases.
- **Large `topK` (20+)** - more context but higher token cost and more noise in the prompt.

## What's Next

- [Chunking Documents](/docs/genai/develop/rag/chunking-documents) -- Chunking strategies for RAG
- [Generating Embeddings](/docs/genai/develop/rag/generating-embeddings) -- Embedding model selection
- [Connecting to Vector Databases](/docs/genai/develop/rag/connecting-vector-dbs) -- Vector store setup
- [What is RAG?](/docs/genai/key-concepts/what-is-rag) -- Conceptual overview
