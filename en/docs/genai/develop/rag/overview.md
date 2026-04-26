---
sidebar_position: 1
title: RAG
description: Reference for the RAG features in WSO2 Integrator — knowledge bases, chunkers, embedding providers, vector stores, data loaders, and the query node.
---

# RAG

**RAG** (Retrieval-Augmented Generation) gives an LLM access to your documents so it can answer questions about them. WSO2 Integrator provides the full pipeline as features in the BI canvas — you wire up a knowledge base, ingest your documents, and call retrieve/augment from any flow.

This section is a feature reference. For an end-to-end walkthrough see the [HR Knowledge Base tutorial](/docs/genai/tutorials).

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [Knowledge Bases](knowledge-bases.md) | The thing that ties together a vector store and an embedding provider. The unit you ingest into and retrieve from. | The right-side **Knowledge Bases** panel on any flow editor. |
| [Embedding Providers](embedding-providers.md) | Connectors that turn text into vectors — Default (WSO2), OpenAI, Azure, Google Vertex, OpenSearch, custom. | **Connections** → AI category, or the **Embedding Provider** field of a Knowledge Base. |
| [Vector Stores](vector-stores.md) | Where the vectors are saved — In-memory, Pinecone, Weaviate, Milvus, pgvector, Azure AI Search. | **Connections** → AI category, or the **Vector Store** field of a Knowledge Base. |
| [Chunker](chunker.md) | Splits documents into chunks before embedding. Auto by default; custom recursive chunker for fine control. | The **Chunker** field of a Knowledge Base, or **Connections** → Generic Recursive Chunker. |
| [Data Loaders](data-loaders.md) | Read documents into memory — Markdown, HTML, PDF, DOCX, PPTX. | **Add Node** → AI → **Data Loader**. |
| [The query nodes](query-node.md) | The flow nodes that retrieve, augment, and call the LLM during a RAG query. | **Add Node** → AI → expand a Knowledge Base for `Ingest`, `Retrieve`, `Delete By Filter`. |

## How the Pieces Fit

```
┌──────────────┐       ┌─────────────┐
│ Data Loader  │  ───► │   Chunker   │ ──┐
└──────────────┘       └─────────────┘   │      ┌───────────────────┐
                                         │  ──► │ Embedding Provider │ ─┐
                                         │      └───────────────────┘   │
                                         │                              ▼
                                         │                     ┌──────────────────┐
                                         └────────────────────►│  Vector Store    │
                                                               └──────────────────┘
                                                                       ▲
                                                                       │
                                                          ┌────────────┴────────────┐
                                                          │   Knowledge Base        │
                                                          │  (vectorStore +         │
                                                          │   embeddingProvider)    │
                                                          └─────────────────────────┘
                                                                       ▲
                                                                       │
                                              ┌─────────┐  ┌──────────┴──────────┐  ┌─────────────────┐
                                              │ Ingest  │  │   Retrieve / Query  │  │ augmentUserQuery│
                                              └─────────┘  └─────────────────────┘  └─────────────────┘
                                                                                              │
                                                                                              ▼
                                                                                       ┌─────────┐
                                                                                       │ chat /  │
                                                                                       │ generate│
                                                                                       └─────────┘
```

A practical project usually has **two** flows:

- An **ingestion flow** (often an automation, a CLI command, or an admin-only HTTP endpoint) that loads, chunks, embeds, and stores documents.
- A **query flow** (typically an HTTP service or an agent tool) that retrieves chunks, augments the user query, and calls the LLM.

The Knowledge Base sits in the middle — the same connection serves both.

## When to Use RAG

| Use RAG when… | Look elsewhere when… |
|---|---|
| You want answers grounded in your own documents (policies, manuals, KB articles, reports). | The data you need is already a structured query away — call your database or API as a tool. |
| Your data changes regularly and you don't want to re-train models. | The dataset is small enough to fit in the prompt directly. Skip the pipeline. |
| You need citation — *"this sentence came from doc X, page Y"*. | The answer is purely about how the model behaves (style, format) — that's a prompt concern, not a data concern. |
| You want to share knowledge across multiple integrations (one Knowledge Base, many flows). | The knowledge is one-off and not reused. |

## What's Next

- **[Knowledge Bases](knowledge-bases.md)** — start here.
- **[Embedding Providers](embedding-providers.md)** and **[Vector Stores](vector-stores.md)** — the two pluggable parts of the knowledge base.
- **[Chunker](chunker.md)** — how documents get split.
- **[Data Loaders](data-loaders.md)** — feeding documents into the pipeline.
- **[Query Nodes](query-node.md)** — using the knowledge base from a flow.
- **[What is RAG?](/docs/genai/key-concepts/what-is-rag)** — conceptual background.
