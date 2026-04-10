---
sidebar_position: 1
title: Chunking Documents
description: Split documents into searchable chunks using the built-in WSO2 Integrator AI chunker or your own custom chunking logic.
---

# Chunking Documents

Chunking determines how your documents are divided into searchable units for a RAG pipeline. The strategy you choose directly impacts retrieval quality -- the right chunks mean the LLM gets exactly the context it needs.

In WSO2 Integrator, chunking is built directly into the `ballerina/ai` module. When you ingest documents into a `VectorKnowledgeBase`, chunking happens automatically. If you need more control, you can pre-chunk documents yourself or disable the built-in chunker entirely.

## Automatic Chunking (Default)

The simplest and most common approach is to let the knowledge base chunk your documents automatically. The default `ai:AUTO` chunker selects a sensible strategy based on the document type.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();

// Third argument defaults to ai:AUTO when omitted
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);

public function main() returns error? {
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();

    // Documents are chunked and embedded automatically during ingestion
    check knowledgeBase.ingest(documents);
    io:println("Ingestion successful");
}
```

The `ai:TextDataLoader` supports `.md`, `.html`, `.pdf`, `.docx`, and `.pptx` files. Pass one or more file paths to load a batch of documents at once.

## Disabling the Built-In Chunker

When you want to feed pre-built chunks directly into the knowledge base without any additional processing, pass `ai:DISABLE` as the third argument to `VectorKnowledgeBase`. This is useful when your content is already structured into chunks or when you need full control over chunk boundaries.

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();

final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider, ai:DISABLE);
```

With `ai:DISABLE`, the knowledge base will embed and store whatever chunks you pass to `ingest` without further splitting.

## Custom Chunking

When you need a specific splitting strategy (fixed-size, paragraph, sentence, or semantic), build the chunks yourself and call `ingest` with an `ai:Chunk[]` value.

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider, ai:DISABLE);

public function main() returns error? {
    string policyText = check io:fileReadString("./leave_policy.md");

    // Example: simple paragraph-based chunking
    string[] paragraphs = re `\n\n+`.split(policyText);

    ai:Chunk[] chunks = from string paragraph in paragraphs
                        where paragraph.trim().length() > 0
                        select {content: paragraph.trim()};

    check knowledgeBase.ingest(chunks);
}
```

The `ai:Chunk` record has a required `content` field that holds the chunk text. Any chunks you produce -- whether from a fixed-size splitter, a semantic chunker, or a third-party library -- can be passed to `ingest` the same way.

### Fixed-Size Chunking Example

```ballerina
function fixedSizeChunks(string text, int maxChars, int overlap) returns ai:Chunk[] {
    ai:Chunk[] chunks = [];
    int i = 0;
    int textLen = text.length();
    while i < textLen {
        int end = int:min(i + maxChars, textLen);
        chunks.push({content: text.substring(i, end)});
        i = end - overlap;
        if i < 0 {
            i = 0;
        }
    }
    return chunks;
}
```

### Contextual Chunking

Prepend document-level context to each chunk to improve retrieval quality for ambiguous queries.

```ballerina
function contextualChunks(string[] rawChunks, string title, string summary) returns ai:Chunk[] {
    return from string chunk in rawChunks
           select {
               content: string `Document: ${title}
Summary: ${summary}
---
${chunk}`
           };
}
```

## Choosing an Approach

| Approach | When to Use |
|---|---|
| **Automatic (`ai:AUTO`)** | Most use cases. Let the framework pick a sensible strategy based on document type. |
| **Disabled (`ai:DISABLE`)** | You already have well-formed chunks from another pipeline or want to feed records directly. |
| **Custom `Chunk[]`** | You need a specific strategy (semantic, sentence, recursive) that the built-in chunker does not cover. |

## What's Next

- [Generating Embeddings](/docs/genai/develop/rag/generating-embeddings) -- Convert chunks to vector embeddings
- [Connecting to Vector Databases](/docs/genai/develop/rag/connecting-vector-dbs) -- Store and query your embeddings
- [RAG Querying](/docs/genai/develop/rag/rag-querying) -- Build the complete query pipeline
