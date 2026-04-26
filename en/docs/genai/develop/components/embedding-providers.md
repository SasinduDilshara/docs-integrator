---
sidebar_position: 3
title: Embedding Providers
description: Reference for every Embedding Provider in WSO2 Integrator BI — init form fields, advanced configurations, defaults, and supported embedding models for the Default WSO2 provider, OpenAI, Azure OpenAI, Google Vertex, and OpenRouter.
---

# Embedding Providers

An **Embedding Provider** turns a chunk of text into a vector. Knowledge Bases use it on **ingest** (to convert chunks into stored vectors) and again on **retrieve** (to convert the user's query into a vector for similarity search).

> **The provider you pick on ingest must be the same one you use on retrieve.** Vectors produced by different providers are not interchangeable — the search will silently return nothing.

## Public Actions

Every embedding provider exposes the same two actions.

| Action | What it does | Required parameters |
|---|---|---|
| **Embed** | Turns a single chunk into a vector. | **Chunk** (the text to embed). |
| **Batch Embed** | Turns many chunks into vectors in one call. Used by Knowledge Bases on bulk ingest. | **Chunks** (the array of chunks). |

You rarely call these directly — Knowledge Base ingest and retrieve calls them for you.

## Where To Find Embedding Providers

Inside the **Create Vector Knowledge Base** form click **+ Create New Embedding Model**, or open the **Embedding Providers** panel from any flow editor. The **Select Embedding Provider** picker shows the supported providers.

![Select Embedding Provider picker listing Default Embedding Provider (WSO2) at the top, then Azure Embedding Provider, Google Vertex Embedding Provider, OpenAI Embedding Provider, and OpenRouter Embedding Provider, each with a one-line description.](/img/genai/develop/components/embedding-providers/01-select-list.png)

## Implementations at a Glance

| Provider | Module | API key required? | Default model |
|---|---|---|---|
| **Default WSO2** | `ballerina/ai` | No (signed-in via WSO2) | WSO2-managed |
| **OpenAI** | [`ballerinax/ai.openai`](https://central.ballerina.io/ballerinax/ai.openai/latest) | Yes | None |
| **Azure OpenAI** | [`ballerinax/ai.azure`](https://central.ballerina.io/ballerinax/ai.azure/latest) | Yes | None |
| **Google Vertex** | [`ballerinax/ai.googleapis.vertex`](https://central.ballerina.io/ballerinax/ai.googleapis.vertex/latest) | OAuth2 / service account | `text-embedding-005` |
| **OpenRouter** | [`ballerinax/ai.openrouter`](https://central.ballerina.io/ballerinax/ai.openrouter/latest) | Yes | None |

> The HTTP-level Advanced Configurations on every external embedding provider use the same set of fields as model providers — see [Model Providers → Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations) for the full reference.

---

## Default WSO2 Embedding Provider

Routes through the WSO2 intelligence service. The same WSO2 sign-in that unlocks the default model provider unlocks this — no separate key required.

### Init form

![Create Embedding Provider form for the Default WSO2 provider. Banner: 'This is a simple operation that requires no parameters. Specify where to store the result to finish.' Two fields: Embedding Provider Name (default aiWso2embeddingprovider) and Result Type (locked to ai:Wso2EmbeddingProvider). Save button.](/img/genai/develop/components/embedding-providers/wso2-default.png)

No provider-specific fields and no Advanced Configurations.

---

## OpenAI

Official: [OpenAI Embeddings docs](https://platform.openai.com/docs/guides/embeddings).

### Init form

![Create Embedding Provider form for OpenAI showing two required fields: API Key and Embedding Model Type. Below: Advanced Configurations Expand link, Embedding Provider Name openaiEmbeddingprovider, Result Type openai:EmbeddingProvider.](/img/genai/develop/components/embedding-providers/openai-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | OpenAI API key. Reference a `configurable` in production. |
| **Embedding Model Type** | Yes | — | `text-embedding-3-small` (1536 dims, configurable down), `text-embedding-3-large` (3072 dims, configurable down), `text-embedding-ada-002` (1536 dims). |

### Advanced configurations

![OpenAI Create Embedding Provider form with Advanced Configurations expanded showing Cache Configuration, Circuit Breaker Configuration, Compression (default AUTO), Forwarded (default 'disable'), HTTP1 Settings, HTTP2 Settings.](/img/genai/develop/components/embedding-providers/openai-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://api.openai.com/v1` | URL string | OpenAI API base URL. |

Plus the [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations).

---

## Azure OpenAI

Official: [Azure OpenAI Embeddings](https://learn.microsoft.com/azure/ai-services/openai/concepts/models#embeddings-models).

### Init form

![Create Embedding Provider form for Azure OpenAI showing four required fields: Access Token, API Version, Deployment ID, Service URL. Below: Advanced Configurations Expand link, Embedding Provider Name azureEmbeddingprovider, Result Type azure:EmbeddingProvider.](/img/genai/develop/components/embedding-providers/azure-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Access Token** | Yes | — | Azure OpenAI API key. |
| **API Version** | Yes | — | Azure OpenAI API version, e.g. `2023-07-01-preview`. |
| **Deployment ID** | Yes | — | Deployment ID for your embedding model deployment. |
| **Service URL** | Yes | — | Base URL of your Azure OpenAI resource, e.g. `https://your-resource.openai.azure.com`. |

> The model name is implicit in the **deployment** on Azure — no Model Type field. Pick the model when you create the deployment in the Azure portal.

### Advanced configurations

![Azure OpenAI Create Embedding Provider form with Advanced Configurations expanded showing Cache Configuration, Circuit Breaker Configuration, Compression AUTO, Forwarded 'disable', HTTP1 Settings, HTTP2 Settings.](/img/genai/develop/components/embedding-providers/azure-advanced.png)

Standard HTTP knobs only — see [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations).

---

## Google Vertex

Official: [Vertex AI Embeddings](https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings).

### Init form

![Create Embedding Provider form for Google Vertex showing two required fields: Auth (record/expression toggle, with hint 'OAuth2RefreshConfig for OAuth2 refresh token flow, or ServiceAccountConfig for automatic token refresh via service account') and Project ID. Below: Advanced Configurations Expand link, Embedding Provider Name vertexEmbeddingprovider, Result Type vertex:EmbeddingProvider.](/img/genai/develop/components/embedding-providers/vertex-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Auth** | Yes | — | OAuth2 refresh-token record, a service-account record, or a path to a service-account JSON file. See [Vertex auth options](/docs/genai/develop/components/model-providers#vertex-auth-options) on the Model Providers page. |
| **Project ID** | Yes | — | Your Google Cloud project ID. |

### Advanced configurations

![Google Vertex Create Embedding Provider form with Advanced Configurations expanded showing Cache Configuration, Circuit Breaker Configuration, Compression AUTO, Forwarded 'disable', HTTP1 Settings, HTTP2 Settings.](/img/genai/develop/components/embedding-providers/vertex-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Location** | `"global"` | `"global"`, `"us-central1"`, `"europe-west1"`, etc. | Google Cloud region. |
| **Model Type** | `text-embedding-005` | `text-embedding-005`, `text-embedding-004`, `textembedding-gecko-multilingual@001`, `textembedding-gecko@001`. | Vertex embedding model. |
| **Service URL** | `""` (auto-derived) | URL string | Override the regional endpoint. Defaults to `https://\{location\}-aiplatform.googleapis.com`. |

Plus the [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations).

---

## OpenRouter

OpenRouter exposes embedding models from many providers behind a single API. Official: [openrouter.ai](https://openrouter.ai/).

### Init form

![Create Embedding Provider form for OpenRouter showing two required fields: API Key (with link to https://openrouter.ai/keys) and Model Type (with example value 'openai/text-embedding-3-small'). Below: Advanced Configurations Expand link, Embedding Provider Name openrouterEmbeddingprovider, Result Type openrouter:EmbeddingProvider.](/img/genai/develop/components/embedding-providers/openrouter-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | OpenRouter API key. |
| **Model Type** | Yes | — | Qualified embedding model name, e.g. `openai/text-embedding-3-small`. See OpenRouter's [model list](https://openrouter.ai/models). |

### Advanced configurations

![OpenRouter Create Embedding Provider form with Advanced Configurations expanded showing Cache Configuration, Circuit Breaker Configuration, Compression AUTO, Forwarded 'disable', HTTP1 Settings, HTTP2 Settings.](/img/genai/develop/components/embedding-providers/openrouter-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://openrouter.ai/api/v1` | URL string | OpenRouter API base URL. |
| **Site URL** | `()` | URL string or empty | Optional site URL sent as `HTTP-Referer`. |
| **Site Name** | `()` | String or empty | Optional site name sent as `X-OpenRouter-Title`. |

Plus the [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations).

> OpenRouter's embedding endpoint accepts text-only chunks. Image or audio chunks raise an error.

---

## Picking a Provider

| Situation | Recommended |
|---|---|
| Prototyping, no infra setup | **Default WSO2** — sign in once, no key. |
| Already on OpenAI for chat | **OpenAI** — same key, same vendor. |
| Already on Azure | **Azure OpenAI** — keep traffic inside your Azure tenant. |
| Already on Google Cloud | **Vertex** — same auth as the rest of GCP. |
| Want one key across many vendors | **OpenRouter**. |

Whatever you pick, **commit to it for the lifetime of the vector store** — you can't mix-and-match without re-embedding everything.

## What's Next

- **[Vector Stores](/docs/genai/develop/components/vector-stores)** — where the embeddings actually live.
- **[Knowledge Bases](/docs/genai/develop/components/knowledge-bases)** — the object that ties an embedding provider, a vector store, and a chunker together.
- **[RAG](/docs/genai/develop/rag/overview)** — the BI canvas walkthrough for ingestion and query flows.
