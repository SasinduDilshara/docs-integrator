---
sidebar_position: 7
title: Query Nodes
description: Reference for the RAG query nodes in WSO2 Integrator — Ingest, Retrieve, Delete By Filter, and ai:augmentUserQuery.
---

# Query Nodes

Once you have a [Knowledge Base](knowledge-bases.md), the right-side panel exposes three actions you can drop into any flow. Combined with `ai:augmentUserQuery` and a `chat` or `generate` node, they form the runtime half of a RAG pipeline.

## Knowledge Base Actions

Expand a Knowledge Base in the right-side panel to see its three actions:

![The Knowledge Bases panel with aiVectorknowledgebase expanded, showing three actions: Ingest, Retrieve, and Delete By Filter.](/img/genai/develop/rag/06-knowledge-base-actions.png)

| Action | When to use |
|---|---|
| **Ingest** | Once per new (or changed) document. Used inside an ingestion flow. |
| **Retrieve** | Every time a user asks a question. The first half of a query. |
| **Delete By Filter** | When documents are removed or replaced. Cleans stale chunks out of the vector store. |

## Ingest

Add an **Ingest** node in your ingestion flow, after your Data Loader.

| Field | What it does |
|---|---|
| **Documents / Chunks** | The thing to ingest — `ai:Document`, `ai:Document[]`, or `ai:Chunk[]`. |
| **Metadata override** | Optional. Extra metadata applied to all chunks ingested in this call. |

What happens behind the scenes:

1. If chunking is enabled, documents are split using the Knowledge Base's chunker.
2. The chunks are embedded by the configured embedding provider (in batch).
3. The vectors and their metadata are written to the vector store.

The action returns when ingestion is complete.

```ballerina
ai:Document|ai:Document[] documents = check loader.load();
check aiVectorKnowledgeBase.ingest(documents);
```

## Retrieve

Add a **Retrieve** node in your query flow, right after the user's question is in scope.

| Field | What it does |
|---|---|
| **Query** | The text to search for — usually the user's question. |
| **Top K** | How many of the closest chunks to return. Common values: 3–10. |
| **Filter** (optional) | Metadata filter to narrow the search (e.g. `doc_type = "policy"`). |

The result is `ai:QueryMatch[]`. Each match has a `chunk` (the original chunk) and a similarity score.

```ballerina
ai:QueryMatch[] matches = check aiVectorKnowledgeBase.retrieve(query, 10);
ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;
```

### Choosing Top K

| Value | Effect |
|---|---|
| 1–3 | Tight, focused answers; fastest; risks missing context. |
| 5–10 | Good default. Enough context without burying the model. |
| 20+ | More context, more tokens, more cost. Mostly noise. |

Start with `5`, raise it only if answers are missing detail, and stop as soon as adding more chunks doesn't improve the answer.

## Augmenting the User Query

Once you have the retrieved chunks, you need to feed them to the LLM together with the user's question. WSO2 Integrator's `ai:augmentUserQuery` builds a standard retrieval-augmented prompt for you:

```ballerina
ai:ChatUserMessage augmented = ai:augmentUserQuery(context, query);
ai:ChatAssistantMessage response = check modelProvider->chat([augmented]);
```

This is the recommended pattern — `augmentUserQuery` produces a prompt that:

- Tells the model to base its answer on the provided context.
- Tells it to say *"I don't know"* if the answer isn't in the context.
- Wraps the chunks and the question in a clear structure.

In a flow, you call `augmentUserQuery` from a small `Call Function` step (the function lives in the `ballerina/ai` module), then send its result into a `chat` node bound to your model provider.

### Inline Style with `generate`

When you want full control of the prompt — for example, a custom "answer in JSON with citations" structure — call `generate` directly with the context interpolated:

```ballerina
string answer = check modelProvider->generate(`Answer based on the following context:

    Context: ${context}

    Query: ${query}

    If the answer is not contained in the context, respond with "I don't know".`);
```

Both styles work; `augmentUserQuery` is the cleaner default for most cases.

## A Complete Query Flow

The typical RAG query resource looks like:

```
Start  ─►  question (read body)
        │
        ▼
   Retrieve(question, topK=5)
        │
        ▼
   augmentUserQuery(context, question)
        │
        ▼
   modelProvider->chat(augmented)
        │
        ▼
    Return  (answer)
```

Generated source:

```ballerina
import ballerina/ai;
import ballerina/http;

service /rag on new http:Listener(8090) {
    resource function post query(@http:Payload QueryRequest req) returns QueryResponse|error {
        ai:QueryMatch[] matches = check aiVectorKnowledgeBase.retrieve(req.question, 5);
        ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;

        if context.length() == 0 {
            return {answer: "I couldn't find any relevant information.", chunksUsed: 0};
        }

        ai:ChatUserMessage augmented = ai:augmentUserQuery(context, req.question);
        ai:ChatAssistantMessage response = check modelProvider->chat([augmented]);

        return {answer: response.content ?: "", chunksUsed: context.length()};
    }
}
```

Test it with curl:

```bash
curl -X POST http://localhost:8090/rag/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the annual leave carry-forward policy?"}'
```

## Delete By Filter

Use **Delete By Filter** in admin/maintenance flows to remove stale vectors:

| Field | What it does |
|---|---|
| **Filter** | A metadata filter that selects which chunks to delete. |

Common patterns:

- **Document version cleanup.** Delete all chunks where `doc_version != "2024-Q4"` after a new version is ingested.
- **Soft delete by tenant.** Delete chunks where `tenantId = "T-1234"` when a tenant offboards.
- **Partial re-ingestion.** Delete by `doc_id`, then ingest the new version.

The exact filter syntax depends on the vector store. Pinecone, Weaviate, Milvus, and pgvector all support metadata filters with slightly different shapes; check the store's connector documentation for specifics.

## Common Mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| `retrieve` returns nothing. | Different embedding provider for ingestion vs query. | Make sure the *same* Knowledge Base (and therefore the same embedding provider) is used in both. |
| `retrieve` returns the wrong document. | Top K too small, or chunks too granular. | Raise Top K, or increase chunk size. |
| Model says "I don't know" when the doc clearly contains the answer. | Chunk lost context (no surrounding paragraph) or wrong filter. | Add contextual chunking (see [Chunker](chunker.md)) or relax the metadata filter. |
| Slow query. | Vector store far from the integration. | Co-locate them. RAG is on the hot path. |
| Stale answers. | Old vectors still present after ingestion. | Use **Delete By Filter** to clean before re-ingesting. |

## Performance and Cost

- **Each query is `embed(question)` + `search(K)` + `LLM(prompt + chunks)`.** The first two are cheap and fast; the LLM call dominates.
- **Cache embeddings of repeat questions.** If your traffic is repetitive (FAQ-style), an in-memory cache of question → embedding pays for itself quickly.
- **Trim chunk content.** A 5-chunk prompt can easily be 3000–5000 tokens. Smaller, well-titled chunks cost less and produce sharper answers.

## What's Next

- **[Knowledge Bases](knowledge-bases.md)** — back to the connection that holds the pipeline together.
- **[Chunker](chunker.md)** — control the input to retrieval.
- **[AI Agents → Tools](/docs/genai/develop/agents/tools)** — add a "search the docs" tool to an agent that wraps a Retrieve call.
