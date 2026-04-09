---
title: "What Is RAG?"
description: "Understand Retrieval-Augmented Generation -- a pattern for grounding LLM responses in your own data."
---

# What Is RAG?

**Retrieval-Augmented Generation (RAG)** is a pattern that grounds LLM responses in your own data instead of relying solely on the model's training knowledge. When a user asks a question, the system first retrieves relevant information from your documents or knowledge base, then sends that information alongside the question to the LLM. The result is an answer that is accurate, up-to-date, and specific to your organization's content.

RAG solves a fundamental limitation of LLMs: they only know what was in their training data, which has a cutoff date and does not include your private documents, internal policies, or domain-specific knowledge. With RAG, you keep the LLM's language abilities while supplementing it with your own data at query time.

## Two Phases of RAG

RAG operates in two distinct phases:

### Ingestion (preparing your data)

During ingestion, you transform raw documents into searchable vector embeddings and store them in a vector database. This is a batch process that runs whenever your source content changes.

**Documents --> Parse --> Chunk --> Embed --> Vector Store**

### Querying (answering questions)

At query time, the user's question follows a retrieve-then-generate flow. The system finds relevant content from your knowledge base and uses it to produce a grounded response.

**Question --> Embed --> Retrieve --> Augment --> Generate --> Answer**

## When to Use RAG

RAG is the right approach when:

- You need answers based on **your own documents** (internal knowledge bases, policies, product docs)
- Your data **changes regularly** and you need responses that reflect current information
- You need **source attribution** -- the ability to point to where an answer came from
- You want to **avoid fine-tuning** costs while still getting domain-specific responses

RAG may not be the best fit when:

- The task is about general knowledge already in the LLM's training data
- You need the model to adopt a specific writing style or personality (fine-tuning is better)
- Your data is small enough to fit entirely in the LLM's context window (direct prompting may suffice)

## In This Section

- [Embeddings and Vector Databases](embeddings-and-vector-databases.md) -- How text becomes searchable vectors
- [RAG Ingestion](rag-ingestion.md) -- The pipeline from documents to stored embeddings
- [RAG Querying](rag-querying.md) -- The flow from question to grounded answer

## What's Next

After understanding RAG concepts, explore the hands-on guides:

- [What Is an AI Agent?](../what-is-an-ai-agent.md) -- Agents can use RAG as part of their reasoning process
- [What Is MCP?](../what-is-mcp/index.md) -- RAG services can be exposed as MCP tools
