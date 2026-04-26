---
sidebar_position: 1
title: AI Connections and Stores
description: Reference for the shared AI connections and stores available in WSO2 Integrator BI — Model Providers, Embedding Providers, Vector Stores, Knowledge Bases, Chunkers, and AI Agent Memory — covering every configuration field, default value, and available option.
---

# AI Connections and Stores

Every AI feature in WSO2 Integrator BI (Direct LLM, Natural Functions, RAG, Agents) is built from the same shared set of **connections and stores**. Pick the one you need, fill the form, and BI generates the connection — you never have to read or write the underlying code.

This section is the reference for every shipped connection and store. For each one you'll find:

- **What it does** in plain words.
- **Where to find it** in the BI canvas (Add Node panel or Artifacts panel).
- **Init parameters** — what to fill in the basic form, with defaults and available values.
- **Advanced configurations** — every optional knob, what it controls, and the default value.
- **Public actions** — the operations the component exposes once it's a connection (e.g., `chat`, `generate`, `ingest`, `retrieve`).

## The Six Connection and Store Types

| Connection / Store | What it does | Reference |
|---|---|---|
| **Model Provider** | Connects to an LLM and exposes `chat` and `generate` actions. Used by Direct LLM, Natural Functions, RAG `generate`, and Agents. | [Model Providers](/docs/genai/develop/components/model-providers) |
| **Embedding Provider** | Turns text chunks into vectors. Used by Knowledge Bases on ingest and on retrieve. | [Embedding Providers](/docs/genai/develop/components/embedding-providers) |
| **Vector Store** | Stores and searches vector embeddings. Backs every Knowledge Base. | [Vector Stores](/docs/genai/develop/components/vector-stores) |
| **Knowledge Base** | The RAG entry point. Ties a Vector Store, an Embedding Provider, and a Chunker together. | [Knowledge Bases](/docs/genai/develop/components/knowledge-bases) |
| **Chunker** | Splits documents into smaller pieces before embedding. | [Chunkers](/docs/genai/develop/components/chunkers) |
| **AI Agent Memory** | Stores per-session chat history for an Agent. | [AI Agent Memory](/docs/genai/develop/components/memory) |

A rough mental model:

- A **Model Provider** drives an LLM.
- A **Knowledge Base** grounds an LLM in your documents and pulls in **Embedding Provider** + **Vector Store** + **Chunker**.
- **Memory** keeps an agent's conversation history across turns.

## Where They Show Up in BI

Every connection is created the same way — open the **Add Node** panel from a flow, expand the **AI** category, and pick the right one.

![Add Node panel with the AI category expanded showing three sub-categories — Direct LLM (Model Provider), RAG (Knowledge Base, Data Loader, Augment Query), Agent — and a tooltip describing model providers as 'Model providers available within the integration for connecting to an LLM'.](/img/genai/develop/components/orientation/03-add-node-direct-llm.png)

| AI sub-category | What you find there |
|---|---|
| **Direct LLM** | Model Provider |
| **RAG** | Knowledge Base, Data Loader, Augment Query |
| **Agent** | Agent |

The **AI Chat Agent** and **MCP Service** artifacts (under **AI Integration** in the Artifacts panel) compose these same connections into ready-made services.

![Artifacts panel with AI Integration section highlighted showing AI Chat Agent and MCP Service options, alongside Integration as API (HTTP, GraphQL, TCP) and Event Integration tiles.](/img/genai/develop/components/orientation/01-artifact-ai-chat-agent.png)

## How To Read These Pages

Each page follows the same shape:

1. **What it does** — one short paragraph.
2. **The list of implementations shipped today.**
3. **Per-implementation sections**, each with:
   - The basic create form, with a screenshot and a parameter table.
   - The advanced configuration view, with a screenshot and a parameter table (defaults + available values).
   - The public actions the connection exposes.

Two universal fields appear on every Create form regardless of the connection or provider:

| Field | What it controls |
|---|---|
| **Name** | The variable name used in the generated source. Pick something descriptive — `emailGenerator`, `supportKb`, `hrChunker`. |
| **Result Type** | The component's type (e.g. `openai:ModelProvider`, `pinecone:VectorStore`). Locked to the provider you picked, so you don't change it by hand. |

These two fields are **not repeated** in the per-provider tables. Everything else listed in those tables is provider-specific.

## What's Next

- **[Model Providers](/docs/genai/develop/components/model-providers)** — start here for Direct LLM, Natural Functions, and Agents.
- **[Embedding Providers](/docs/genai/develop/components/embedding-providers)** + **[Vector Stores](/docs/genai/develop/components/vector-stores)** — the building blocks of a [Knowledge Base](/docs/genai/develop/components/knowledge-bases).
- **[Chunkers](/docs/genai/develop/components/chunkers)** — only matters when AUTO chunking misses your document structure.
- **[AI Agent Memory](/docs/genai/develop/components/memory)** — only matters for Agents.
