---
title: "RAG Ingestion"
description: "Understand the ingestion phase of RAG -- how documents are transformed into searchable vector embeddings."
---

# RAG Ingestion

The ingestion phase is where you prepare your data for retrieval. Raw documents are transformed through a multi-step pipeline into vector embeddings stored in a database, ready to be searched at query time. You build this pipeline once and run it whenever your source content changes to keep your knowledge base current.

## The Ingestion Pipeline

The ingestion pipeline follows a consistent sequence of steps:

```
Documents (PDF, HTML, Markdown, text)
        |
        v
    [ Parse ]     -- Extract raw text from source formats
        |
        v
    [ Chunk ]     -- Split text into meaningful segments
        |
        v
    [ Embed ]     -- Convert chunks into vector embeddings
        |
        v
   [ Store ]      -- Persist embeddings in a vector database
```

Each step transforms the data into a form that is closer to what the retrieval system needs. Let's examine each step.

### Step 1: Parse -- Extract text from documents

Your source content may come in various formats: PDF reports, HTML pages, Markdown documentation, plain text files, or even database records. The first step extracts the raw text content from these formats, stripping away formatting markup while preserving the meaningful structure.

WSO2 Integrator provides data loaders for common formats. For example, a text data loader reads Markdown or plain text files and wraps them into document records that the rest of the pipeline can process.

### Step 2: Chunk -- Split text into segments

A single document is usually too large to embed as one unit. Chunking splits documents into smaller, meaningful segments that can be individually embedded and retrieved. The goal is to create chunks that are:

- **Small enough** to be specific and relevant when retrieved
- **Large enough** to contain complete thoughts and useful context

Common chunking strategies include:

| Strategy | How it works | Best for |
|---|---|---|
| **Fixed-size** | Split every N characters | Uniform content like logs or FAQs |
| **Recursive character** | Split on natural boundaries (paragraphs, sentences) before falling back to character limits | Most document types (recommended default) |
| **Sentence-based** | Group N sentences per chunk | Conversational content, transcripts |
| **Semantic** | Use an embedding model to detect topic shifts and split at semantic boundaries | Long-form content with distinct sections |

**Overlap** between chunks ensures that context at boundaries is not lost. A typical configuration uses 10% overlap -- for a 500-character chunk, 50 characters overlap with the next chunk.

### Step 3: Embed -- Convert to vectors

Each chunk is passed through an embedding model, which converts the text into a dense numerical vector. This vector captures the semantic meaning of the chunk in a form that enables mathematical similarity comparison.

The embedding model and vector dimensions must be consistent across your entire pipeline. If you use a model that produces 1536-dimension vectors during ingestion, you must use the same model (or one with the same dimensions) during querying.

### Step 4: Store -- Persist in a vector database

The final step stores each vector alongside its original text content and any metadata (source file, page number, section title, timestamp) in a vector database. Metadata is not part of the vector similarity calculation, but it enables filtered search and source attribution during retrieval.

## Ingestion in Ballerina

WSO2 Integrator provides a streamlined API for building ingestion pipelines. Here is an example using the official `ai` module:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);

public function main() returns error? {
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();
    check knowledgeBase.ingest(documents);
}
```

In this example:

- `ai:TextDataLoader` handles parsing the source file.
- `ai:VectorKnowledgeBase` manages the chunking, embedding, and storage steps internally.
- A single call to `knowledgeBase.ingest(documents)` runs the full pipeline.

## Key Design Decisions

When building an ingestion pipeline, consider:

- **Chunk size** -- Smaller chunks (200-500 tokens) give precise retrieval; larger chunks (500-1000 tokens) preserve more context. Start with 500 characters and adjust based on retrieval quality.
- **Overlap** -- 10% overlap is a good default. Increase for dense technical content where boundary context matters.
- **Embedding model** -- Choose based on quality, speed, and cost requirements. 1536-dimension models are the standard starting point.
- **Update strategy** -- Decide whether to re-index everything on each run or track changes and only process updated documents.

## What's Next

- [RAG Querying](rag-querying.md) -- How the query phase retrieves context and generates grounded answers
- [Embeddings and Vector Databases](embeddings-and-vector-databases.md) -- Deeper look at how embeddings and vector search work
