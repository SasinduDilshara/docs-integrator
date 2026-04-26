---
sidebar_position: 2
title: "Tutorial: RAG Knowledge Base"
description: Build a RAG-powered knowledge base in WSO2 Integrator that ingests enterprise documents and answers questions using retrieval-augmented generation.
---

# Tutorial: RAG Knowledge Base

The end-to-end **RAG knowledge base** tutorial is now maintained as **[Building an HR Knowledge Base with RAG](/docs/genai/tutorials/hr-knowledge-base-rag)**. It walks through the same pipeline (Data Loader → Vector Knowledge Base → ingest, then retrieve → augment → generate) using the BI canvas and the built-in `ai` module — no third-party SDK code required.

If you landed here from an older bookmark:

- For a hands-on RAG tutorial, follow **[Building an HR Knowledge Base with RAG](/docs/genai/tutorials/hr-knowledge-base-rag)**.
- For the conceptual reference (components, ingestion flow, query flow, tuning), see **[RAG](/docs/genai/develop/rag/overview)**.
- For per-component options (vector stores, embedding providers, chunkers), see **[AI Connections and Stores](/docs/genai/develop/components/overview)**.

## What's Next

- **[Building an HR Knowledge Base with RAG](/docs/genai/tutorials/hr-knowledge-base-rag)** — the canonical end-to-end RAG tutorial.
- **[RAG → Tuning Retrieval](/docs/genai/develop/rag/overview#tuning-retrieval)** — Top K, filters, and other knobs.
- **[AI Connections and Stores → Vector Stores](/docs/genai/develop/components/vector-stores)** — swap In-Memory for Pinecone, Milvus, pgvector, or Weaviate.
- **[AI Customer Support Agent](ai-customer-support.md)** — reuse a Vector Knowledge Base from a chat agent as a tool.
- **[Multi-Agent Workflow](multi-agent-workflow.md)** — combine RAG with other agents in a larger workflow.
