---
sidebar_position: 4
title: Vector Stores
description: Reference for every Vector Store in WSO2 Integrator BI — init form fields, advanced configurations, query modes, and metadata filter support for In-Memory, Pinecone, pgvector, Weaviate, and Milvus.
---

# Vector Stores

A **Vector Store** persists vector embeddings and answers similarity queries over them. It's the storage half of a [Knowledge Base](/docs/genai/develop/components/knowledge-bases) — the [Embedding Provider](/docs/genai/develop/components/embedding-providers) produces vectors, the Vector Store keeps them and finds the nearest matches at query time.

## Public Actions

Every vector store exposes the same three actions. You don't usually call them directly — Knowledge Base ingest, retrieve, and `deleteByFilter` use them under the hood.

| Action | What it does | Required parameters |
|---|---|---|
| **Add** | Persists vector entries (embeddings + their source chunks). Replaces existing entries with the same id. | **Entries** (the vectors to add). |
| **Query** | Returns the most similar entries for a given query embedding and/or metadata filter. | **Query** (an embedding and/or filters, plus `topK`). |
| **Delete** | Deletes entries by id. | **IDs** (a single id or list). |

### Query input fields

When something calls **Query** on a vector store, the request carries these fields:

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Embedding** | optional | A vector | The vector to use for similarity search. If omitted, the search returns by filter only. |
| **Filters** | optional | Metadata filters | Restrict the search by metadata fields. See [Metadata filters](#metadata-filters). |
| **Top K** | `10` | Any positive integer or `-1` (all) | Max number of items to return. |

### Metadata filters

Most stores support filtering vectors by their metadata using standard operators:

| Operator | Meaning |
|---|---|
| `==` | Equal |
| `!=` | Not equal |
| `>` `<` `>=` `<=` | Greater than / less than (and equal) |
| `in` | Value is in the given list |
| `nin` | Value is not in the given list |

Multiple filters can be combined with `AND` or `OR`. The exact wire-format mapping (Pinecone uses `$eq`, pgvector compiles to JSONB, Weaviate uses GraphQL `Equal`, Milvus has its own filter syntax) is handled by each connector — you write filters the same way regardless of store.

### Query modes

Some stores support more than just dense vector search. The mode you pick when you create the store determines what kind of embeddings it accepts:

| Mode | When to use | Supported by |
|---|---|---|
| `DENSE` | Standard semantic search using dense vectors. The default everywhere. | All stores |
| `SPARSE` | Keyword/lexical-style search using sparse vectors. | Pinecone, pgvector |
| `HYBRID` | Combine dense and sparse vectors. | Pinecone |

### Similarity metrics

Local stores let you choose the metric. Hosted stores manage it themselves (you pick when you create the index/collection in their UI).

| Metric | Measures |
|---|---|
| `COSINE` | Cosine of the angle between vectors. Most common for semantic search. |
| `EUCLIDEAN` | Straight-line distance between vector points. |
| `DOT_PRODUCT` | Directional similarity, magnitude-sensitive. Not supported on pgvector. |
| `MANHATTAN` | Sum of absolute differences (pgvector only). |

## Where To Find Vector Stores

Inside the **Create Vector Knowledge Base** form click **+ Create New Vector Store**, or open the **Vector Stores** panel from any flow editor. The **Select Vector Store** picker shows the supported stores:

![Select Vector Store picker listing In Memory Vector Store, Milvus Vector Store, Pgvector Vector Store, Pinecone Vector Store (highlighted), and Weaviate Vector Store, each with a one-line description.](/img/genai/develop/components/vector-stores/01-select-list.png)

## Implementations at a Glance

| Store | Module | Modes supported | Hosted/local |
|---|---|---|---|
| **In-Memory** | `ballerina/ai` | DENSE | Local (process memory) |
| **Pinecone** | [`ballerinax/ai.pinecone`](https://central.ballerina.io/ballerinax/ai.pinecone/latest) | DENSE, SPARSE, HYBRID | Hosted |
| **pgvector** | [`ballerinax/ai.pgvector`](https://central.ballerina.io/ballerinax/ai.pgvector/latest) | DENSE, SPARSE | Self-hosted PostgreSQL |
| **Weaviate** | [`ballerinax/ai.weaviate`](https://central.ballerina.io/ballerinax/ai.weaviate/latest) | DENSE | Hosted or self-hosted |
| **Milvus** | [`ballerinax/ai.milvus`](https://central.ballerina.io/ballerinax/ai.milvus/latest) | DENSE | Hosted or self-hosted |

---

## In-Memory Vector Store

Embeddings live in the running integration's process memory. **Not durable** — data is lost on restart. Useful for development, tests, and tiny datasets.

### Init form

![Create Vector Store form for In-Memory showing the banner 'This operation has no required parameters. Optional settings can be configured below.' Advanced Configurations Expand link, Vector Store Name (default aiInmemoryvectorstore), Result Type (locked to ai:InMemoryVectorStore).](/img/genai/develop/components/vector-stores/in-memory-basic.png)

No required fields.

### Advanced configurations

![In-Memory Vector Store Create form with Advanced Configurations expanded showing Similarity Metric (default COSINE).](/img/genai/develop/components/vector-stores/in-memory-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Similarity Metric** | `COSINE` | `COSINE`, `EUCLIDEAN`, `DOT_PRODUCT` | Metric used for vector similarity. |

> Supports dense vectors only. Adding sparse or hybrid vectors raises an error.

---

## Pinecone

Hosted vector database with native dense, sparse, and hybrid support. Multi-tenant via namespaces. Official: [docs.pinecone.io](https://docs.pinecone.io).

### Init form

![Create Vector Store form for Pinecone showing two required fields: API Key (Pinecone API key for authentication) and Service URL (URL of the Pinecone API service). Below: Advanced Configurations Expand link, Vector Store Name pineconeVectorstore, Result Type pinecone:VectorStore.](/img/genai/develop/components/vector-stores/pinecone-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Pinecone API key. |
| **Service URL** | Yes | — | URL of the Pinecone index endpoint. |

### Advanced configurations

![Pinecone Vector Store Create form with Advanced Configurations expanded showing Pinecone Configuration (default {}), HTTP Configuration (default {}), Query Mode (default ai:DENSE).](/img/genai/develop/components/vector-stores/pinecone-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Pinecone Configuration** | `{}` | Record (`namespace`, `filters`, `sparseVector`) | Pinecone-specific settings. **Namespace** isolates vectors for multi-tenancy. **Filters** sets default metadata filters applied on every query. **Sparse Vector** is needed for hybrid search. |
| **HTTP Configuration** | `{}` | Record | Standard HTTP knobs — same fields as [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations). |
| **Query Mode** | `ai:DENSE` | `ai:DENSE`, `ai:SPARSE`, `ai:HYBRID` | Search mode. |

> `topK` must be in the range 1–10000.

---

## pgvector

Vector search inside PostgreSQL using the pgvector extension. The connector creates the table and an HNSW index automatically on first use. Official: [pgvector on GitHub](https://github.com/pgvector/pgvector).

### Init form

![Create Vector Store form for pgvector showing four required fields: Database Name, Host Name, Password, Username. Below: Advanced Configurations Expand link, Vector Store Name pgvectorVectorstore, Result Type pgvector:VectorStore.](/img/genai/develop/components/vector-stores/pgvector-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Database Name** | Yes | — | PostgreSQL database name. |
| **Host Name** | Yes | — | Database host (e.g. `localhost`). |
| **Password** | Yes | — | Database password. |
| **Username** | Yes | — | Database user. |

### Advanced configurations

![pgvector Vector Store Create form with Advanced Configurations expanded showing Configurations For The Vector Store (default {}), Properties To Configure Connection Pool (default {}), Additional Set Of Configurations For The Database (default {}), Port Number (default 5432), Table Name (default 'vector_store').](/img/genai/develop/components/vector-stores/pgvector-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Configurations For The Vector Store** | `{}` | Record (`embeddingType`, `vectorDimension`, `similarityMetric`) | **Embedding Type**: `ai:DENSE` (default) or `ai:SPARSE` — picks the column type (`vector` vs `sparsevec`). **Vector Dimension**: `1536` by default; must match your embedding provider's output dimension. **Similarity Metric**: `COSINE` (default), `EUCLIDEAN`, or `MANHATTAN`. |
| **Properties To Configure Connection Pool** | `{}` | Record | Connection pool settings. |
| **Additional Set Of Configurations For The Database** | `{}` | Record | Extra PostgreSQL options (SSL mode, connect timeout, etc.). |
| **Port Number** | `5432` | Any positive integer | Database port. |
| **Table Name** | `"vector_store"` | String | Table to store vectors in. Created on first use if missing. |

> Auto-created table schema: `id VARCHAR PRIMARY KEY, content TEXT, embedding vector|sparsevec, metadata JSONB`. An HNSW index is created automatically for fast similarity search.

---

## Weaviate

Open-source vector database with structured filtering and a GraphQL query layer. The connector queries pre-existing collections — you create the collection (with its schema) in Weaviate first. Official: [weaviate.io/docs](https://weaviate.io/developers/weaviate).

### Init form

![Create Vector Store form for Weaviate showing three required fields: API Key (The API key for the Weaviate service), Weaviate Configuration (record/expression toggle, default '\{collectionName: ""\}'), and Service URL. Below: Advanced Configurations Expand link, Vector Store Name weaviateVectorstore, Result Type weaviate:VectorStore.](/img/genai/develop/components/vector-stores/weaviate-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Weaviate API key (sent as a bearer token). |
| **Weaviate Configuration** | Yes | `\{collectionName: ""\}` | Record with collection-level config. **Collection Name** (required): the Weaviate collection to use; must already exist. **Chunk Field Name** (optional, default `"content"`): the field on the collection that holds the chunk content. |
| **Service URL** | Yes | — | The Weaviate endpoint URL. |

### Advanced configurations

![Weaviate Vector Store Create form with Advanced Configurations expanded showing HTTP Configuration (default {}).](/img/genai/develop/components/vector-stores/weaviate-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **HTTP Configuration** | `{}` | Record | Standard HTTP knobs — same fields as [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations). |

> Currently dense vectors only. Weaviate's `certainty` score is mapped to the `similarityScore` field in the response.

---

## Milvus

Open-source vector database optimised for very large datasets. Like Weaviate, the collection (and its schema and index) must exist before the connector talks to it. Official: [milvus.io/docs](https://milvus.io/docs).

### Init form

![Create Vector Store form for Milvus showing three required fields: API Key (The API key for the Milvus service), Milvus Configuration (record/expression toggle, default {}), and Service URL. Below: Advanced Configurations Expand link, Vector Store Name milvusVectorstore, Result Type milvus:VectorStore.](/img/genai/develop/components/vector-stores/milvus-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Milvus API key (sent as a bearer token). |
| **Milvus Configuration** | Yes | `{}` | Record with collection settings. **Collection Name** (default `"default"`): the Milvus collection to use. **Chunk Field Name** (optional): the field on the collection that holds the chunk content. **Primary Key Field** (default `"id"`): the collection's primary-key field. **Additional Fields** (default `[]`): extra fields to include in search results, on top of `content`, `type`, `vector`, `metadata`. |
| **Service URL** | Yes | — | The Milvus service URL. |

### Advanced configurations

![Milvus Vector Store Create form with Advanced Configurations expanded showing HTTP Configuration (default {}).](/img/genai/develop/components/vector-stores/milvus-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **HTTP Configuration** | `{}` | Record | Standard HTTP knobs — same fields as [Standard HTTP Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations). |

> The collection is loaded into memory automatically before each search. IDs are converted to integers for Milvus's primary key field.

---

## Picking a Store

| Situation | Recommended |
|---|---|
| Prototyping; tiny dataset; tests | **In-Memory** — zero infra. |
| Already running PostgreSQL | **pgvector** — keep vectors next to your existing data. |
| Want hosted, multi-tenant by default | **Pinecone**. |
| Want open-source plus rich filtering & GraphQL | **Weaviate**. |
| Very large datasets, k8s-native | **Milvus**. |

Pick on operational concerns (where your data already lives, what your team already runs). All five satisfy the same Vector Store contract — the rest of the project doesn't change when you swap.

## What's Next

- **[Knowledge Bases](/docs/genai/develop/components/knowledge-bases)** — combine a vector store with an embedding provider and a chunker.
- **[Embedding Providers](/docs/genai/develop/components/embedding-providers)** — what produces the vectors the store will hold.
- **[RAG](/docs/genai/develop/rag/overview)** — the BI canvas walkthrough for ingestion and query.
