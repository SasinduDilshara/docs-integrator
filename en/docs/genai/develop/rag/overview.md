---
sidebar_position: 1
title: RAG
description: Single-page learning reference for Retrieval-Augmented Generation in WSO2 Integrator, the components, the ingestion flow, the query flow, and the configuration choices.
---

# RAG

**RAG** (Retrieval-Augmented Generation) gives an LLM access to your documents so it can answer questions about them. WSO2 Integrator provides the full pipeline as features in the BI canvas.

This is a learning reference. It walks through how the pieces fit together and what each one does, in the order you'd typically encounter them in a project. It is not a tutorial; there's no copy-paste recipe, instead every section explains what BI is doing for you and what choices you have.

## When to Use RAG

| Use RAG when… | Look elsewhere when… |
|---|---|
| You want answers grounded in your own documents (policies, manuals, KB articles, reports). | The data you need is already a structured query away, call your database or API as a tool. |
| Your data changes regularly and you don't want to re-train models. | The dataset is small enough to fit in the prompt directly. Skip the pipeline. |
| You need citation, *"this sentence came from doc X, page Y"*. | The answer is purely about how the model behaves (style, format), that's a prompt concern, not a data concern. |
| You want to share knowledge across multiple integrations (one Knowledge Base, many flows). | The knowledge is one-off and not reused. |

## How the Pieces Fit

Before looking at any canvas, it helps to have the mental model. RAG in BI is a small set of components arranged around a single central abstraction, the **Knowledge Base**.

```
┌──────────────┐       ┌─────────────┐
│ Data Loader  │  ───► │   Chunker   │ ──┐
└──────────────┘       └─────────────┘   │      ┌────────────────────┐
                                         │  ──► │ Embedding Provider │ ─┐
                                         │      └────────────────────┘  │
                                         │                              ▼
                                         │                     ┌──────────────────┐
                                         └────────────────────►│   Vector Store   │
                                                               └──────────────────┘
                                                                       ▲
                                                                       │
                                                          ┌────────────┴────────────┐
                                                          │   Knowledge Base        │
                                                          │  (vectorStore +         │
                                                          │   embeddingProvider +   │
                                                          │   chunker)              │
                                                          └─────────────────────────┘
                                                                       ▲
                                                                       │
                                              ┌─────────┐  ┌──────────┴──────────┐  ┌─────────────────┐
                                              │ Ingest  │  │   Retrieve / Query  │  │ augmentUserQuery│
                                              └─────────┘  └─────────────────────┘  └─────────────────┘
                                                                                              │
                                                                                              ▼
                                                                                       ┌─────────┐
                                                                                       │generate │
                                                                                       └─────────┘
```

The **Knowledge Base** is the central abstraction. It owns three things:

- A **Vector Store**, where the embeddings live (in-memory for development; Pinecone, Weaviate, Milvus, pgvector, or Azure AI Search for production).
- An **Embedding Provider**, what turns text into vectors (Default WSO2, OpenAI, Azure, Google Vertex, OpenSearch, or a custom one).
- A **Chunker**, how documents are split before embedding (AUTO by default, or a custom recursive chunker for fine control).

Two more nodes flank it:

- A **Data Loader** sits before ingestion, it reads documents from disk into memory so the Knowledge Base can ingest them.
- The **query nodes** (`ai:retrieve`, `ai:augmentUserQuery`, `ai:generate`) sit after retrieval, they pull chunks out of the Knowledge Base and feed them to the LLM.

## Anatomy of a RAG Project

A practical RAG project has **two flows** that share a single Knowledge Base.

- An **ingestion flow** loads documents, chunks and embeds them, and stores the result. It typically lives inside an Automation (or an admin-only HTTP endpoint) and runs once per document set.
- A **query flow** runs on every user request: it retrieves the most relevant chunks, augments the user's question with them, and asks the LLM to answer. It typically lives inside an HTTP service (or an agent tool).

Below is what those two flows look like on the BI canvas after they're built. They're presented up-front because the rest of this page explains how each node in them works.

**Ingestion flow** (in an Automation):

![A BI automation flow on the canvas: Start → ai:load (aiDocumentAlDocument) → ai:ingest → log:printInfo with message 'Ingestion Completed!' → Error Handler. Left sidebar shows Connections including aiVectorknowledgebase, aiWso2embeddingprovider, aiWso2modelprovider, milvusVectorStore.](/img/genai/develop/rag/19-ingestion-flow-full.png)

**Query flow** (in an HTTP service resource):

![A BI HTTP-service resource flow for POST /query in the Sample-Integration project: Start → ai:retrieve (queryMatch) → ai:augmentUserQuery (aiChatusermessage) → ai:generate (result, branched out to a small aiWso2modelprov node on the right) → Return (result) → Error Handler. Left sidebar shows Connections aiInmemoryvectorstore, aiVectorknowledgebase, aiWso2embeddingprovider, aiWso2modelprovider.](/img/genai/develop/rag/20-query-flow-full.png)

The shared piece between them is the **Knowledge Base**, the connection in the project's Connections tree. Ingestion writes to it; the query flow reads from it. Build the Knowledge Base once, and any flow in the project can use it.

The rest of this page walks through building each piece in the order you'd touch it in a real project: first the Knowledge Base and its three plug-in parts, then the ingestion flow, then the query flow.

---

## Building the Knowledge Base

The **Vector Knowledge Base** is the connection that ties a vector store, an embedding provider, and a chunker together. You typically create it once per project from the **Knowledge Bases** panel (which opens when you add a node that needs a Knowledge Base, like `ai:ingest`):

![The ai:Vector Knowledge Base configuration panel with Vector Store set to 'aiInmemoryvectorstore', Embedding Model set to 'aiWso2embeddingprovider', Chunker set to AUTO (default), Knowledge Base Name set to 'aiVectorknowledgebase', and Result Type set to 'ai:VectorKnowledgeBase'.](/img/genai/develop/rag/22-vector-knowledge-base-filled.png)

Three pluggable parts:

| Part | What it is | Common choices |
|---|---|---|
| **Vector Store** | Where the embeddings are persisted. The Knowledge Base will store vectors here on ingest and search them on retrieve. | **In-Memory** (good for dev), **Pinecone**, **Weaviate**, **Milvus**, **pgvector**. See [Components → Vector Stores](/docs/genai/develop/components/vector-stores). |
| **Embedding Model** | The function that turns text into a vector. **Must be the same provider on ingest and on retrieve**, otherwise nothing matches. | **Default WSO2** (no key, signs in once), **OpenAI**, **Azure OpenAI**, **Google Vertex**, **OpenRouter**, custom. See [Components → Embedding Providers](/docs/genai/develop/components/embedding-providers). |
| **Chunker** | How a document gets split before embedding. Smaller chunks improve retrieval precision; larger chunks preserve context. | **AUTO** (chooses based on document type), **DISABLE**, or a structure-aware chunker (Generic Recursive, Markdown, HTML, Devant). See [Components → Chunkers](/docs/genai/develop/components/chunkers). |

The **+ Create New** link next to each field lets you create a fresh connection inline if you don't already have one. Each pluggable part is a connection in its own right and can be reused across multiple Knowledge Bases. The next three sub-sections walk through what creating each part looks like.

### Creating a Vector Store

Click **+ Create New Vector Store** in the Knowledge Base form (or open the **Vector Stores** panel from any flow editor and click **+**). The **Select Vector Store** picker slides in with one card per supported store:

![The Select Vector Store side panel listing five vector stores: In Memory Vector Store (with description 'An in-memory vector store implementation that provides simple storage for vector entries'), Milvus Vector Store, Pgvector Vector Store, Pinecone Vector Store (highlighted), and Weaviate Vector Store. Each card has a one-line description.](/img/genai/develop/rag/08-select-vector-store.png)

Pick the store you want and its **Create Vector Store** form opens. The fields depend on the store; a hosted vector DB like Milvus needs a service URL, an API key, and the per-collection configuration:

![The Create Vector Store form for Milvus with required fields: API Key* (with Text/Expression toggle), Milvus Configuration* (Record/Expression toggle, default '{}'), Service URL* (Text/Expression toggle). Advanced Configurations section below shows HTTP Configuration. Vector Store Name* set to 'milvusVectorstore' and Result Type* set to 'milvus:VectorStore' at the bottom.](/img/genai/develop/rag/07-create-vector-store-milvus.png)

The fields you fill differ by store, but always include a connection name and a type. For the per-store init parameter reference (Pinecone, pgvector, Weaviate, Milvus, In-Memory) see [Components → Vector Stores](/docs/genai/develop/components/vector-stores).

For development, the **In-Memory Vector Store** has no fields, just a name and a type. Pick it when you're getting a flow working and switch to a hosted store later by editing the connection.

### Creating an Embedding Provider

Click **+ Create New Embedding Model** in the Knowledge Base form, or open the **Embedding Providers** panel from any flow editor. The **Select Embedding Provider** picker shows the supported providers:

![The Select Embedding Provider side panel listing Default Embedding Provider (WSO2) at the top with a 'Default' badge, then Azure OpenAi Embedding Provider, Google Vertex Embedding Provider, OpenAi Embedding Provider, OpenSearch Embedding Provider. Each entry has a one-line description.](/img/genai/develop/rag/03-select-embedding-provider.png)

The **Default Embedding Provider (WSO2)** form is the simplest, just a name and a type, no API key. The same WSO2 sign-in that unlocks the default model provider unlocks the default embedding provider:

![The Create Embedding Provider form for the Default WSO2 provider. Header: 'This is a simple operation that requires no parameters. Specify where to store the result to finish.' Two fields: Embedding Provider Name* (default 'aiWso2embeddingprovider') and Result Type* (default 'ai:Wso2EmbeddingProvider', locked). Save button.](/img/genai/develop/rag/09-create-embedding-provider-default.png)

For external providers, the form takes an API key (or service-account auth, for Google Vertex) plus an embedding model name. The full per-provider parameter reference is in [Components → Embedding Providers](/docs/genai/develop/components/embedding-providers).

**The provider you pick on ingest must be the same one used on retrieve.** The vectors produced by different providers are not compatible, so reusing the same Embedding Provider connection across the ingestion and query flows is what keeps RAG working.

### Creating a Chunker

A chunker decides how each document is split into pieces before being embedded. Most projects start with the default (**AUTO**) and only create a custom chunker when retrieval quality drops because chunks are too big or too small.

Click **+ Create New Chunker** in the Knowledge Base form. The **Select Chunker** picker shows the available chunker types:

![The Select Chunker side panel listing three chunkers: Generic Recursive Chunker (description 'Represents a Generic document chunker. Provides functionality to recursively chunk a text…'), Markdown Chunker, and Html Chunker.](/img/genai/develop/rag/11-select-chunker.png)

Pick a chunker and its **Create Chunker** form opens. The Generic Recursive Chunker has no required fields, the defaults are sensible; you only open **Advanced Configurations** when you need to tune chunk size or overlap:

![The Create Chunker form for a Generic Recursive Chunker. Header: 'This operation has no required parameters. Optional settings can be configured below.' Advanced Configurations section is collapsed (Expand link). Chunker Name* set to 'aiGenericrecursivechunker' and Result Type* set to 'ai:GenericRecursiveChunker' at the bottom. Save button.](/img/genai/develop/rag/04-create-chunker.png)

The two universal fields (**Chunker Name** and **Result Type**) appear on every chunker form. Under **Advanced Configurations** each chunker exposes its own knobs — `maxChunkSize`, `maxOverlapSize`, and a strategy enum that varies by chunker type. The full reference for each chunker, its strategies, and the `chunkDocumentRecursively` / `chunkMarkdownDocument` / `chunkHtmlDocument` convenience functions is in [Components → Chunkers](/docs/genai/develop/components/chunkers).

The **Markdown** and **Html** chunkers are structure-aware variants, they split along headings or tags first, falling back to character-based splits within sections. Use them when your documents have meaningful structure that AUTO doesn't preserve.

When the Knowledge Base is configured with **AUTO** (the default), BI picks one of these chunkers internally based on the document type. Creating a chunker connection and assigning it explicitly only matters when you want to override that choice or configure non-default settings.

---

## The Ingestion Flow

With the Knowledge Base in place, the ingestion flow's job is to push your documents into it. It's typically an **Automation** so it can be invoked manually or scheduled.

### The Data Loader

The Data Loader reads documents from one or more paths and produces an array of `ai:Document` values. You add it from the **Data Loaders** panel — expand a created loader (here `textDocumentLoader`) and click **Load** to drop a load node into the Automation:

![The Data Loaders side panel inside an Automation. The textDocumentLoader entry is expanded and exposes a single action 'Load — Loads documents as `TextDocument`s from a source.' The cursor is hovering over the Load action; the canvas shows Start → 'Select node from node panel' placeholder → Error Handler.](/img/genai/develop/rag/26-data-loader-load-action.png)

Once added, the load node's configuration takes the path(s). The path itself is almost always a `configurable` so the same flow can ingest different folders in different environments.

![The ai:Data Loader configuration panel with a Paths field set to a template literal `${path}` (a reference to a configurable variable), an Add New Item link to add more paths, a Data Loader Name field set to 'textDocumentLoader', and a Result Type field set to 'ai:TextDataLoader'.](/img/genai/develop/rag/21-data-loader-configured.png)

Loaders ship for the common document formats (Markdown, HTML, PDF, DOCX, PPTX). The result type tells you which loader you got, `ai:TextDataLoader` for text-based formats, with format-specific variants for the others. Pass a directory path (not individual files) to let BI batch reads.

### Ingesting Documents

Once the Knowledge Base exists, expanding it in the **Knowledge Bases** side panel surfaces three actions, **Ingest**, **Retrieve**, and **Delete By Filter**. Inside the ingestion Automation, hover **Ingest** to see its description before clicking:

![The Knowledge Bases side panel inside an Automation. aiVectorknowledgebase is expanded and exposes three actions — Ingest (highlighted with the description 'Indexes a collection of chunks. Converts each chunk to an embedding and stores it in the vector store, making the chunk searchable through the retriever.'), Retrieve, and Delete By Filter. The canvas to the left shows ai:load (hrDocuments) followed by a 'Select node from node panel' placeholder.](/img/genai/develop/rag/06-knowledge-base-actions.png)

**Ingest** is the action you put in the ingestion flow. It takes the array of documents from the Data Loader, hands them to the configured chunker, embeds each chunk through the Embedding Provider, and writes the resulting vectors to the Vector Store.

![The ai:ingest configuration panel showing a Documents* field bound to the variable 'hrDocuments' (the result of the Data Loader).](/img/genai/develop/rag/12-ai-ingest-node.png)

The other two actions (**Retrieve** and **Delete By Filter**) belong on the query flow and on a separate housekeeping flow respectively; **Delete By Filter** is how you clean up old chunks before re-ingesting an updated version of a document.

### Putting the Ingestion Flow Together

After Data Loader and Ingest are in place, the automation looks like this on the canvas:

![A BI automation flow: Start → ai:load (hrDocuments) → ai:ingest → log:printInfo 'Ingestion Completed!' → Error Handler.](/img/genai/develop/rag/19-ingestion-flow-full.png)

The `log:printInfo` step at the end is optional but useful, you'll want to see in the run log when ingestion has actually completed before kicking off queries.

---

## The Query Flow

A query flow runs on every user request: retrieve, augment, generate, return. It typically lives in an HTTP service resource.

### Retrieve

The **Knowledge Base → Retrieve** action queries the Knowledge Base for the most relevant chunks for the user's question. It's the read-side counterpart to **Ingest**, same Knowledge Base, opposite direction. Inside the query HTTP resource, expand the Knowledge Base in the **Knowledge Bases** side panel and pick **Retrieve**:

![The Knowledge Bases side panel inside the POST /query HTTP resource. aiVectorknowledgebase is expanded and exposes three actions — Ingest, Retrieve (cursor hovering, just before click), and Delete By Filter. The canvas to the left shows Start followed by a 'Select node from node panel' placeholder where the retrieve node will land.](/img/genai/develop/rag/25-kb-retrieve-action.png)

After clicking **Retrieve**, the configuration panel asks for the user's query and a result variable. Bind the Query field to the incoming question and store the matches in `queryMatch`:

![The retrieve configuration panel showing a Query* field bound to the user's question, a Result* field set to 'queryMatch', and a Result Type* of ai:QueryMatch[].](/img/genai/develop/rag/13-ai-retrieve-node.png)

The result is an array of `ai:QueryMatch` values, each with a chunk plus its similarity score. This is what you'd hand to a custom prompt if you wanted to cite-and-quote, but most flows hand it straight to the next node.

### Augment Query

`ai:augmentUserQuery` is the bridge between RAG and the LLM. It takes the retrieved chunks (the *context*) and the user's original question (the *query*) and produces an `ai:ChatUserMessage` that bundles them in a format the LLM understands.

![The ai:augmentUserQuery configuration panel with Context* set to queryMatch, Query* set to userQuery, Result* set to aiChatusermessage, and Result Type* set to ai:ChatUserMessage.](/img/genai/develop/rag/16-ai-augmentuserquery-node.png)

You don't need to write a prompt that interleaves chunks and the question yourself, this node does it. The output is one message; pass it directly to `ai:generate`.

### Generate

`ai:generate` calls the LLM with the augmented message. The Prompt field references `${aiChatusermessage.content}` so the model sees the chunks plus the question, and the **Expected Type** field shapes the response (see [Direct LLM → Binding Typed Responses](/docs/genai/develop/direct-llm/overview#binding-typed-responses)).

![A BI HTTP-service resource flow showing Start → ai:retrieve → ai:augmentUserQuery, with the right-side panel showing aiWso2modelprovider → generate. The Prompt field uses the content of the augmented chat user message; Result and Expected Type fields are below.](/img/genai/develop/rag/23-rag-generate-augmented-prompt.png)

This is the same `generate` node documented in [Direct LLM Calls](/docs/genai/develop/direct-llm/overview#the-generate-node), the only difference is the prompt gets its body from `aiChatusermessage` rather than a hand-written template.

### Return

A plain Return step sends the LLM's output back to the HTTP caller. With type binding on `generate`, what you return is already the right Ballerina type; no JSON parsing required.

### Putting the Query Flow Together

The completed query resource is short:

![A BI HTTP-service resource flow for POST /query in the Sample-Integration project: Start → ai:retrieve (queryMatch) → ai:augmentUserQuery (aiChatusermessage) → ai:generate (result, branched to aiWso2modelprov on the right) → Return (result) → Error Handler. Left sidebar shows Connections aiInmemoryvectorstore, aiVectorknowledgebase, aiWso2embeddingprovider, aiWso2modelprovider.](/img/genai/develop/rag/20-query-flow-full.png)

Five nodes, all standard BI building blocks. The RAG-specific work (retrieving the right chunks and presenting them to the model) is encapsulated in the first two nodes; the last two are the same `generate` and `Return` you'd use without RAG.

---

## At the Project Level

Both flows live in the same project and reference the same Connections. The project's Design view makes that visible:

![The Sample-Integration project Design overview in WSO2 Integrator. The left Connections tree lists aiInmemoryvectorstore, aiVectorknowledgebase, aiWso2embeddingprovider, aiWso2modelprovider. The main canvas wires httpDefaultListener and the Automation entry point through to the /api/v1 http:Service (with POST /query) and out to aiWso2embeddingprovider (Embedding Provider) and aiWso2modelprovider (Model Provider).](/img/genai/develop/rag/24-rag-project-design.png)

A few things worth noticing:

- The **Knowledge Base** appears once and is shared. The ingestion automation and the query HTTP service both bind to it.
- The **Embedding Provider** is a separate connection because it can also be used outside RAG (for example, a future feature that does similarity matching directly).
- The **Vector Store** is also a separate connection, which lets you swap In-memory for Pinecone in production without touching the Knowledge Base's other settings.

This separation is what lets RAG scale across artifacts without re-creating connections.

---

## Component Reference

The four pluggable parts of a RAG project — Knowledge Base, Vector Store, Embedding Provider, Chunker — each have a dedicated component reference page with init parameters, advanced configurations, and supported implementations:

| Component | What's on the reference page |
|---|---|
| **[Knowledge Bases](/docs/genai/develop/components/knowledge-bases)** | The `ai:KnowledgeBase` abstract type, the default `ai:VectorKnowledgeBase`, the `ingest` / `retrieve` / `deleteByFilter` API, the `augmentUserQuery` helper, and the document and metadata types. |
| **[Vector Stores](/docs/genai/develop/components/vector-stores)** | In-Memory, Pinecone, pgvector, Weaviate, Milvus — init params, configuration records, advanced HTTP knobs, and supported query modes (DENSE / SPARSE / HYBRID) and similarity metrics. |
| **[Embedding Providers](/docs/genai/develop/components/embedding-providers)** | Default WSO2, OpenAI, Azure OpenAI, Google Vertex, OpenRouter — init params and supported models. |
| **[Chunkers](/docs/genai/develop/components/chunkers)** | `AUTO`, `DISABLE`, `GenericRecursiveChunker`, `MarkdownChunker`, `HtmlChunker`, the strategy enums, and the convenience functions. |

A few RAG-specific knobs are worth knowing about up front:

- **Top K** (on `retrieve`) — how many chunks to pull. Default is `10`. Raise it when relevant content gets missed; pass `-1` to return everything.
- **Filter** (on `retrieve` and on `deleteByFilter`) — restrict matches by metadata using `ai:MetadataFilters`. See [Vector Stores → Metadata filters](/docs/genai/develop/components/vector-stores#metadata-filters).
- **HTTP-client and timeout settings** — the same fields as a model provider; see [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations).

Whatever embedding provider you pick on **ingest** must match what's used on **retrieve**, the vector spaces are not interchangeable.

---

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| `retrieve` returns nothing on a query you know matches a document. | Different embedding provider used for ingestion vs. query. | The same Knowledge Base must be used for both. Check the **Embedding Model** field on your Knowledge Base. |
| Retrieve returns the wrong document, or a related-but-wrong section. | Top K too small, or chunks too granular. | Raise Top K (try 10), or move from AUTO chunking to a structure-aware chunker. |
| LLM responds *"I don't know"* even when the document clearly has the answer. | Chunk lost surrounding context, or metadata filter is too strict. | Add contextual chunking (prepend doc title/summary to each chunk), or relax the filter. |
| Answers contradict each other depending on phrasing. | Two retrieved chunks contradict because they're from related sections. | Use larger chunks, or filter to a single document version. |
| Slow query (multi-second). | Vector store far from the integration, or Top K too high. | Co-locate them; reduce Top K. |
| Stale answers after updating a document. | Old chunks still in the vector store. | Use **Delete By Filter** to remove old chunks before re-ingesting. |
| Default embedding provider not signed in. | First-run setup not completed. | Run **Ballerina: Configure default WSO2 model provider** from the Command Palette. |
| Ingestion takes forever on a large folder. | Loading file-by-file. | Pass the directory path (not individual files) to the Data Loader so BI can batch reads. |

## What's Next

- **[Direct LLM Calls](/docs/genai/develop/direct-llm/overview)**, the `generate` node and Expected Type binding used at the end of the query flow.
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)**, package a RAG-augmented prompt as a typed reusable function.
- **[AI Agents](/docs/genai/develop/agents/overview)**, wrap retrieval as an agent tool when the task needs multi-step reasoning.
- **[What is RAG?](/docs/genai/key-concepts/what-is-rag)**, conceptual background.
