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

The **+ Create New** link next to each field lets you create a fresh connection inline. Each pluggable part is a connection in its own right and can be reused across multiple Knowledge Bases.

> The form fields, defaults, and advanced configurations for each part live on the dedicated component pages — [Vector Stores](/docs/genai/develop/components/vector-stores), [Embedding Providers](/docs/genai/develop/components/embedding-providers), [Chunkers](/docs/genai/develop/components/chunkers), and the Knowledge Base itself on [Knowledge Bases](/docs/genai/develop/components/knowledge-bases). They are not duplicated here.

A few rules worth remembering when wiring a Knowledge Base:

- **Use the same Embedding Provider on ingest and retrieve.** Vectors from different providers are not interchangeable; if you change the provider, you must re-ingest everything.
- **In-Memory is fine for development**; switch to a hosted Vector Store (Pinecone, Weaviate, Milvus, pgvector) for production. The Knowledge Base only needs the connection swapped — your flows don't change.
- **Leave the Chunker on `AUTO`** until retrieval quality drops. AUTO picks the right chunker (Markdown / HTML / Generic Recursive) based on document type.


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

## Tuning Retrieval

Two knobs make the biggest difference at query time:

- **Top K** (on `retrieve`) — how many chunks to pull. Default is `10`. Raise it when relevant content gets missed; pass `-1` to return everything (be careful — this can return a lot of data).
- **Filters** (on `retrieve` and on `deleteByFilter`) — restrict matches by metadata. Useful for multi-tenant projects where each user should only see their own documents. See [Vector Stores → Metadata filters](/docs/genai/develop/components/vector-stores#metadata-filters) for the operator reference.

For deep changes — different vector store, different embedding model, custom chunker, hybrid search — see the relevant [Components](/docs/genai/develop/components/overview) page.

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
