---
sidebar_position: 2
title: Knowledge Bases
description: Reference for the Knowledge Base feature in WSO2 Integrator — the connection that ties together a vector store and an embedding provider for RAG.
---

# Knowledge Bases

A **Knowledge Base** is the central RAG connection. It bundles a vector store, an embedding provider, and a chunking strategy into one artifact that flows ingest into and retrieve from. Once configured, it shows up as a connection in your project and as a category in the right-side panel of any flow editor.

## The Knowledge Bases Panel

Open any flow editor and look on the right. The **Knowledge Bases** panel is one of the categories in the side panel. It lists every knowledge-base connection in the project, plus a button to add a new one.

![The Knowledge Bases side panel listing two implementations: Vector Knowledge Base ('Represents a vector knowledge base for managing chunk indexing and retrieval...') and Azure AI Search Knowledge Base ('Represents the Azure Search Knowledge Base implementation.').](/img/genai/develop/rag/01-knowledge-bases-panel.png)

Two kinds of Knowledge Base ship with WSO2 Integrator:

| Type | Module | Use when… |
|---|---|---|
| **Vector Knowledge Base** | `ballerina/ai` | You want full control of the pipeline. Pick your own vector store, embedding provider, and chunker. The most common choice. |
| **Azure AI Search Knowledge Base** | `ballerinax/ai.azure` | You're already running Azure AI Search and want to use its built-in chunking, vectorisation, and retrieval. |

This page focuses on the Vector Knowledge Base; the Azure version uses the same flow nodes and only differs in configuration.

## Creating a Vector Knowledge Base

Click **+** at the top of the Knowledge Bases panel and pick **Vector Knowledge Base**. The configuration form opens:

![The Vector Knowledge Base configuration form. Fields visible: Vector Store (Default field with Select / Expression toggle), Embedding Model, Chunker (default 'AUTO'), Knowledge Base Name (default value visible), Result Type. + Create New Vector Store and + Create New Embedding Provider links visible. Save button.](/img/genai/develop/rag/02-vector-knowledge-base-config.png)

| Field | Required | What it does |
|---|---|---|
| **Vector Store** | Yes | The connection where vectors are stored. Pick an existing one or **+ Create New Vector Store**. |
| **Embedding Model** | Yes | The connection that turns text into vectors. Pick an existing one or **+ Create New Embedding Provider**. |
| **Chunker** | No | How documents are split. Default `AUTO` works for most projects. See [Chunker](chunker.md). |
| **Knowledge Base Name** | Yes | The variable name in generated code. |
| **Result Type** | Yes | The Ballerina type — `ai:VectorKnowledgeBase` for the standard option. |

Both the Vector Store and the Embedding Provider can be created inline from this form — the **+ Create New** links open the relevant picker on the right and add the new connection back into the form when you save it.

## What BI Generates

After saving, BI writes one connection block in your project:

```ballerina
import ballerina/ai;

final ai:VectorStore aiInmemoryVectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider aiWso2EmbeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase aiVectorKnowledgeBase =
        new ai:VectorKnowledgeBase(aiInmemoryVectorStore, aiWso2EmbeddingProvider);
```

You don't write this; the form does it for you. But the underlying API is small enough that source view is readable — and switching pieces (say, swapping `InMemoryVectorStore` for a Pinecone store) is a one-line change.

## Knowledge Base Actions

Once a Knowledge Base exists, the right-side panel shows the actions you can call on it. Expand the connection in the panel and the action buttons appear:

![The Knowledge Bases panel with aiVectorknowledgebase expanded, showing three actions: Ingest, Retrieve, and Delete By Filter.](/img/genai/develop/rag/06-knowledge-base-actions.png)

| Action | What it does |
|---|---|
| **Ingest** | Feed documents (or pre-built chunks) in. Triggers chunking, embedding, and writing to the vector store. |
| **Retrieve** | Find the chunks that best match a query. Returns ranked matches with their source metadata. |
| **Delete By Filter** | Remove vectors that match a metadata filter — for example, delete all chunks from a specific document version. |

These are the verbs you drop into your flows. See [Query Nodes](query-node.md) for the full picture.

## One Knowledge Base, Many Flows

A Knowledge Base is a **project-scoped connection**. The same connection can be:

- Used by an HTTP service that exposes a `/query` endpoint.
- Used by an automation that re-ingests documents on a schedule.
- Used as a tool by an [AI Agent](/docs/genai/develop/agents/overview).
- Used by a natural function that needs grounded context.

You don't duplicate the configuration anywhere. Add the connection once, and every flow that needs it picks it from the panel.

## Picking the Right Pieces

The default **Vector Knowledge Base** with **Default Embedding Provider (WSO2)** and **In-Memory Vector Store** is a great starting point — zero configuration, no external services. Move to a real vector store and a different embedding provider when:

| Reason | Switch to… |
|---|---|
| You want vectors to survive process restarts. | Any persistent vector store (Pinecone, Weaviate, Milvus, pgvector, Azure AI Search). |
| You need vectors to be queryable from outside this integration. | A managed vector store the rest of your platform can hit. |
| You're concerned about data residency. | A self-hosted store (pgvector on your own DB; Milvus on your own cluster). |
| You want a non-WSO2 embedding model. | OpenAI, Azure, Google Vertex, or a custom provider. |
| Your queries need hybrid search (keyword + vector). | Weaviate or Azure AI Search. |

The pluggable design is the point — you can change one piece without touching the rest of the pipeline.

## What's Next

- **[Embedding Providers](embedding-providers.md)** — pick the model that turns text into vectors.
- **[Vector Stores](vector-stores.md)** — pick where the vectors live.
- **[Chunker](chunker.md)** — control how documents get split.
- **[Query Nodes](query-node.md)** — call ingest / retrieve / delete from a flow.
