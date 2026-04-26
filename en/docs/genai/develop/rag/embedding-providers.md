---
sidebar_position: 3
title: Embedding Providers
description: Reference for embedding providers in WSO2 Integrator — Default WSO2, OpenAI, Azure, Google Vertex, OpenSearch, and custom providers.
---

# Embedding Providers

An **embedding provider** turns a chunk of text into a vector — a list of numbers that capture the chunk's meaning. Vectors with similar meaning end up near each other in number-space, which is what makes semantic search possible.

WSO2 Integrator ships with several embedding providers and lets you plug in your own. Same interface for all of them, so you can switch with one config change.

## The Select Embedding Provider Panel

When you create a Knowledge Base or click the **Embedding Model** field, BI opens the **Select Embedding Provider** picker:

![The Select Embedding Provider panel listing five providers: Default Embedding Provider (WSO2) (highlighted as default), Azure Embedding Provider, Google Vertex Embedding Provider, OpenAI Embedding Provider, and OpenSearch Embedding Provider.](/img/genai/develop/rag/03-select-embedding-provider.png)

| Provider | Module | Notes |
|---|---|---|
| **Default Embedding Provider (WSO2)** | `ballerina/ai` | Pre-configured with your WSO2 account. No API key. The fastest path to a working RAG pipeline. |
| **OpenAI Embedding Provider** | `ballerinax/ai.openai` | `text-embedding-3-small`, `text-embedding-3-large`. Highest quality at varying cost/dimension trade-offs. |
| **Azure Embedding Provider** | `ballerinax/ai.azure` | OpenAI embedding models hosted on Azure. Required for Azure data-residency. |
| **Google Vertex Embedding Provider** | `ballerinax/ai.googlevertex` | Vertex AI text-embedding models. |
| **OpenSearch Embedding Provider** | `ballerinax/ai.opensearch` | If you use OpenSearch as your search platform, leverage its embedding pipeline. |

After picking one, the configuration form opens. Common fields:

| Field | What it does |
|---|---|
| **API key** (or `configurable` reference) | Required for non-WSO2 providers. Always store via `Config.toml`, never inline. |
| **Model name** | The embedding model to use (e.g. `text-embedding-3-small`, `embedding-001`). Defaults are sensible. |
| **Dimensions** | Some providers let you reduce dimensionality. Default is the model's natural size. |

Click **Save** and the provider becomes a connection in your project — pick it from the **Embedding Model** field of your Knowledge Base.

## What BI Generates

For the default provider:

```ballerina
final ai:EmbeddingProvider aiWso2EmbeddingProvider = check ai:getDefaultEmbeddingProvider();
```

For OpenAI:

```ballerina
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:EmbeddingProvider openAiEmbedding = check new openai:EmbeddingProvider(
    apiKey = openAiApiKey,
    modelType = openai:TEXT_EMBEDDING_3_SMALL
);
```

You only ever write the configurable line by hand (and that's optional — BI generates a stub for you). The connection itself comes from the form.

## The Golden Rule: Use the Same Provider Everywhere

**Vectors from different models are not comparable.** A vector from OpenAI's `text-embedding-3-small` is in a different space from a vector from the WSO2 default provider, even if they are about the same sentence.

That means:

- Use the same provider during **ingestion** and **retrieval**. The query gets embedded by the same model that embedded the documents.
- If you change the provider, **re-ingest** all your documents.
- A Knowledge Base only references one provider; it cannot mix.

If you ever need to migrate, the cleanest path is: build a new Knowledge Base with the new provider, ingest documents into it, swap the reference in your flow, then drop the old Knowledge Base.

## Picking Between Providers

| Concern | Pick |
|---|---|
| Fastest to start, no keys, prototyping. | **Default WSO2**. |
| Highest accuracy, you have an OpenAI account. | **OpenAI** with `text-embedding-3-large`. |
| Azure data residency. | **Azure**. |
| Lowest cost at scale, reasonable accuracy. | **OpenAI** with `text-embedding-3-small` (or its smaller dimensions). |
| Already on Vertex / GCP. | **Google Vertex**. |
| You operate OpenSearch and want it integrated. | **OpenSearch**. |
| Embedding model not in the list. | Custom provider — see below. |

In practice, accuracy differences between modern providers are smaller than chunking and prompt-quality differences. Don't over-optimise the embedding choice before the rest of the pipeline is solid.

## Batching

The embedding provider's `batchEmbed` method lets you embed many chunks in a single API call. The default Vector Knowledge Base uses batching automatically during `ingest`, so you don't need to think about it. If you write a custom provider, implement `batchEmbed` for performance and cost reasons.

## Custom Embedding Providers

For embedding models not covered by the built-in connectors, implement the `ai:EmbeddingProvider` interface in a class of your own:

```ballerina
import ballerina/ai;

isolated client class MyEmbeddingProvider {
    *ai:EmbeddingProvider;

    isolated remote function embed(ai:Chunk chunk) returns ai:Embedding|ai:Error {
        // Call your embedding service and return the vector
        return [0.1, 0.2, 0.3, /* ... */ ];
    }

    isolated remote function batchEmbed(ai:Chunk[] chunks)
            returns ai:Embedding[]|ai:Error {
        // Batch implementation for efficiency
        return from ai:Chunk c in chunks select <ai:Embedding>[0.1, 0.2, 0.3];
    }
}
```

Pass an instance to your `VectorKnowledgeBase` and the rest of the pipeline works unchanged.

## Cost Awareness

Embeddings are paid by the token. Two practical levers:

- **Cheap models for prototyping.** WSO2 default and OpenAI's small model are inexpensive — use them while you're iterating.
- **Re-ingest only when you must.** Embedding costs are paid every time you re-ingest, so avoid unnecessary churn — for example, don't re-ingest everything when only one document changed; use **Delete By Filter** plus targeted ingestion.

## What's Next

- **[Vector Stores](vector-stores.md)** — where the vectors get saved.
- **[Knowledge Bases](knowledge-bases.md)** — wire the embedding provider into the pipeline.
- **[Query Nodes](query-node.md)** — use the resulting Knowledge Base in a flow.
