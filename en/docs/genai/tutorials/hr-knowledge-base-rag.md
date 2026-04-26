---
sidebar_position: 1
title: "Building an HR Knowledge Base with RAG"
description: Step-by-step tutorial for building an HR knowledge base in WSO2 Integrator that ingests HR policy documents and answers employee questions over HTTP using retrieval-augmented generation.
---

# Building an HR Knowledge Base with RAG

**Time:** 30 minutes&nbsp;&nbsp;|&nbsp;&nbsp;**Level:** Intermediate&nbsp;&nbsp;|&nbsp;&nbsp;**What you'll build:** Two artifacts in a single integration — an Automation that ingests a folder of HR policy documents into a vector knowledge base, and an HTTP Service that answers employee questions with retrieval-augmented generation.

In this tutorial you build a complete HR RAG pipeline visually in WSO2 Integrator's flow designer. The Automation walks documents → chunker → embedding model → vector store. The HTTP Service walks employee question → retrieve → augment → generate → response. No third-party SDK code is required — everything is wired with the built-in `ai` module nodes.

## Prerequisites

- [WSO2 Integrator installed](/docs/get-started/install) and signed into [WSO2 Integrator Copilot](/docs/genai/getting-started/setup) — Copilot provisions the default model and embedding providers; the first time you use them, BI prompts you to run **Ballerina: Configure default WSO2 model provider** from the Command Palette and writes the credentials into `Config.toml` automatically.
- A folder of HR policy documents in plain-text form (leave policy, benefits, code of conduct, onboarding, etc.). A few short `.txt` files are enough to follow the tutorial — content does not have to be production-ready.

## Architecture

```
Ingestion (Automation):
folder path → Data Loader → ai:load → ai:ingest → Vector Knowledge Base
                                                  ├── Vector Store (In-Memory)
                                                  ├── Embedding Provider (WSO2)
                                                  └── Chunker (AUTO)

Query (HTTP Service POST /api/v1/query):
employee question → ai:retrieve → ai:augmentUserQuery → ai:generate → JSON response
```

## Step 1: Open the Integration

Open or create an integration project in WSO2 Integrator (this tutorial uses **Sample-Integration**). The empty integration view shows an **+ Add Artifact** button — that's your starting point for the whole tutorial.

![Empty integration overview](/img/genai/tutorials/hr-knowledge-base-rag/01-empty-integration.png)

Click **+ Add Artifact**. The artifact catalog opens, grouping the artifact types by category.

![Add Artifact catalog](/img/genai/tutorials/hr-knowledge-base-rag/02-add-artifact-catalog.png)

You will create two artifacts in this catalog: an **Automation** (under *Automation*) for ingestion, then an **HTTP Service** (under *Integration as API*) for querying.

## Step 2: Create the Ingestion Automation

### 2.1 Pick the Automation artifact

Click the **Automation** card. WSO2 Integrator opens the **Create New Automation** dialog — accept the defaults and click **Create**.

![Create New Automation](/img/genai/tutorials/hr-knowledge-base-rag/03-create-new-automation.png)

The automation flow editor opens with an empty `Start` node and an `Error Handler` end node. Click the **+** between them to open the node palette.

The palette groups every node type — **Statement** (Declare/Update Variable, Call Function, Map Data), **Control** (If, Match, While, Foreach, Return), **AI** (Direct LLM, RAG, Agent), and so on.

![Automation node palette](/img/genai/tutorials/hr-knowledge-base-rag/04-automation-node-palette.png)

### 2.2 Add a Data Loader

Under **AI → RAG**, click **Data Loader**. The **Data Loaders** picker lists the available loader types. Pick **Text Data Loader**.

![Data Loaders picker](/img/genai/tutorials/hr-knowledge-base-rag/05-data-loaders-picker.png)

The **ai : Data Loader** side panel opens. BI suggests a default **Data Loader Name** (e.g. `aiTextdataloader`); rename it to `textDocumentLoader` for this tutorial so later screenshots match. **Result Type** stays at the auto-filled `ai:TextDataLoader`.

![Data Loader form — initial state](/img/genai/tutorials/hr-knowledge-base-rag/06-data-loader-form-empty.png)

Click into the **Paths** field to start entering the source folder. Select **+ New Configurable** to externalise the folder path so you can change it without editing the flow. Fill in:

- **Variable Name** — `path`
- **Variable Type** — `string`
- **Documentation** — *Path for the HR documents to ingest.*

![New Configurable for path](/img/genai/tutorials/hr-knowledge-base-rag/07-new-configurable-path.png)

Save the configurable and complete the Data Loader form:

- **Paths** — `` string `${path}` `` (uses the configurable)
- **Data Loader Name** — `textDocumentLoader`
- **Result Type** — `ai:TextDataLoader`

![Data Loader form filled](/img/genai/tutorials/hr-knowledge-base-rag/08-data-loader-form-filled.png)

Click **Save**.

### 2.3 Add the `ai : load` Node

Click the **+** below the Data Loader. The **Data Loaders** panel opens and lists the `textDocumentLoader` connection you just created with its **Load** action — *"Loads documents as `TextDocument`s from a source."* Hover and click **Load**.

![Load action just before click](/img/genai/tutorials/hr-knowledge-base-rag/09a-load-action-picker.png)

The **ai : load** form opens. Set:

- **Result** — `hrDocuments`
- **Result Type** — `ai:Document[] | ai:Document` (auto-filled)

![ai:load form filled](/img/genai/tutorials/hr-knowledge-base-rag/09-ai-load-form-filled.png)

### 2.4 Create the Vector Knowledge Base

Click **+** below `ai:load` and choose **Knowledge Base** under **AI → RAG**. The **Knowledge Bases** picker shows the supported types — pick **Vector Knowledge Base**.

![Knowledge Bases picker](/img/genai/tutorials/hr-knowledge-base-rag/10-knowledge-bases-picker.png)

The **ai : Vector Knowledge Base** form opens with all fields empty. It has three required building blocks: **Vector Store**, **Embedding Model**, and **Chunker**. Each can be created inline.

![Empty Vector Knowledge Base form](/img/genai/tutorials/hr-knowledge-base-rag/16a-vector-knowledge-base-empty.png)

Build them one at a time.

#### 2.4.1 Create the Vector Store

Click **+ Create New Vector Store**. The picker lists the supported stores: **In Memory**, **Milvus**, **Pgvector**, **Pinecone**, **Weaviate**.

![Select Vector Store](/img/genai/tutorials/hr-knowledge-base-rag/11-vector-store-picker.png)

Pick **In Memory Vector Store** — no external infra required for this tutorial. The **Create Vector Store** form opens. Fill in:

- **Vector Store Name** — `aiInmemoryvectorstore`
- **Result Type** — `ai:InMemoryVectorStore` (auto-filled)

![Create Vector Store filled](/img/genai/tutorials/hr-knowledge-base-rag/12-create-vector-store-filled.png)

Click **Save**. The new connection appears in the left **Connections** tree.

#### 2.4.2 Create the Embedding Provider

Back on the Vector Knowledge Base form, click **+ Create New Embedding Model**. The picker lists the supported providers: **Default Embedding Provider (WSO2)**, **Azure**, **Google Vertex**, **OpenAI**, **OpenRouter**.

![Select Embedding Provider](/img/genai/tutorials/hr-knowledge-base-rag/13-embedding-provider-picker.png)

Pick **Default Embedding Provider (WSO2)** — it is provisioned through your Copilot login, no API key required. The **Create Embedding Provider** form opens. Fill in:

- **Embedding Provider Name** — `aiWso2embeddingprovider`
- **Result Type** — `ai:Wso2EmbeddingProvider` (auto-filled)

![Create Embedding Provider filled](/img/genai/tutorials/hr-knowledge-base-rag/14-create-embedding-provider-filled.png)

Click **Save**.

#### 2.4.3 Pick (or Create) a Chunker

Back on the Vector Knowledge Base form, the **Chunker** field defaults to `AUTO` — the runtime selects a chunker based on document type. For most HR text documents this is fine.

If you want to control chunking explicitly, click **+ Create New Chunker**. The picker offers:

- **Generic Recursive Chunker** — recursive splitting with configurable `maxChunkSize` and `maxOverlapSize`.
- **Markdown Chunker** — preserves markdown structure.
- **Html Chunker** — preserves HTML structure.

![Select Chunker](/img/genai/tutorials/hr-knowledge-base-rag/15-chunker-picker.png)

For this tutorial, leave the chunker at `AUTO`.

#### 2.4.4 Save the Knowledge Base

The Vector Knowledge Base form is now fully populated:

- **Vector Store** — `aiInmemoryvectorstore`
- **Embedding Model** — `aiWso2embeddingprovider`
- **Chunker** — `AUTO`
- **Knowledge Base Name** — `aiVectorknowledgebase`
- **Result Type** — `ai:VectorKnowledgeBase`

![Vector Knowledge Base filled](/img/genai/tutorials/hr-knowledge-base-rag/16-vector-knowledge-base-filled.png)

Click **Save**. The left **Connections** tree now lists `aiInmemoryvectorstore`, `aiWso2embeddingprovider`, and `aiVectorknowledgebase` — these are reusable from any artifact in the project.

### 2.5 Add the `ai : ingest` Node

Click **+** below the previous node. The **Knowledge Bases** panel opens and lists `aiVectorknowledgebase` with its actions — **Ingest**, **Retrieve**, **Delete By Filter**. Hover **Ingest** — the description reads *"Indexes a collection of chunks. Converts each chunk to an embedding and stores it in the vector store, making the chunk searchable through the retriever."* Click it.

![Ingest action just before click](/img/genai/tutorials/hr-knowledge-base-rag/17a-ingest-action-picker.png)

The **ai : ingest** form opens. The **Documents** field expects an array of `Document` records. The expression editor exposes the record shape so you can confirm each chunk has the required fields:

![Ingest record configuration](/img/genai/tutorials/hr-knowledge-base-rag/17-ingest-record-configuration.png)

Set **Documents** to your loaded array `hrDocuments` and click **Save**.

![ai:ingest filled](/img/genai/tutorials/hr-knowledge-base-rag/18-ai-ingest-filled.png)

### 2.6 Log Completion (Optional)

Click **+** below `ai:ingest` and pick **Log → printInfo** under **Logging**. Set the message to `"Ingestion Completed!"` so you can confirm in the run log when ingestion has actually finished — useful before kicking off any queries against the store.

### 2.7 Review the Completed Ingestion Flow

Your ingestion automation now contains:

```
Start → ai:load (hrDocuments) → ai:ingest → log:printInfo → Error Handler
```

![Completed automation flow](/img/genai/tutorials/hr-knowledge-base-rag/19-automation-flow-complete.png)

Run the automation once (▶ **Run**) — it loads every file under `path`, chunks it, embeds it, and populates `aiInmemoryvectorstore`. Back on the project overview, you can see the Automation artifact connected to its embedding provider:

![Project overview after automation](/img/genai/tutorials/hr-knowledge-base-rag/20-project-after-automation.png)

:::tip In-Memory store is volatile
Restart the runtime and you must re-ingest. For a persistent store, swap the vector store for Pinecone, Milvus, Pgvector, or Weaviate; the rest of the flow stays the same.
:::

## Step 3: Create the Query HTTP Service

### 3.1 Add the HTTP Service Artifact

Go back to the project overview and click **+ Add Artifact** again. Under **Integration as API**, pick **HTTP Service**.

In the configuration screen, accept the default **Listener** (`httpDefaultListener`) and set **Base Path** to `/api/v1`.

![HTTP Service configured](/img/genai/tutorials/hr-knowledge-base-rag/21-http-service-config.png)

### 3.2 Add the `query` Resource

Click **+ Add Resource** and fill in:

- **HTTP Method** — `POST`
- **Resource Path** — `query`
- **Define Payload** — accept JSON (the body becomes `userQuery`)
- **Responses** — `201` returning `json`, `500` returning `error`

![Add Resource configuration](/img/genai/tutorials/hr-knowledge-base-rag/22-add-resource-form.png)

Click **Save** and open the resource flow.

### 3.3 Retrieve Relevant Chunks

Click the **+** in the resource flow. The **Knowledge Bases** panel opens with `aiVectorknowledgebase` (the same connection you created in the Automation) and its three actions. Click **Retrieve**.

![Retrieve action just before click](/img/genai/tutorials/hr-knowledge-base-rag/22a-retrieve-action-picker.png)

The **ai : retrieve** form opens — it runs a vector search against your HR knowledge base. Fill in:

- **Query** — `userQuery` (the request payload)
- **Result** — `queryMatch`
- **Result Type** — `ai:QueryMatch[]` (auto-filled)

![ai:retrieve filled](/img/genai/tutorials/hr-knowledge-base-rag/23-ai-retrieve-filled.png)

Click **Save**.

### 3.4 Augment the User Query

Click **+** below `ai:retrieve` and pick **`ai : augmentUserQuery`**. It packages the employee's question together with the retrieved chunks into a `ChatUserMessage` ready for the LLM — no manual prompt-templating required. Fill in:

- **Context** — `[queryMatch]` (the array of matches from the previous node)
- **Query** — `userQuery` (the employee's original question)
- **Result** — `aiChatusermessage`
- **Result Type** — `ai:ChatUserMessage` (auto-filled)

![ai:augmentUserQuery filled](/img/genai/tutorials/hr-knowledge-base-rag/24-ai-augment-filled.png)

Click **Save**.

### 3.5 Add a Model Provider

In the AI section of the palette, click **Model Provider** and add the WSO2 default provider. Fill in:

- **Model Provider Name** — `aiWso2modelprovider`
- **Result Type** — `ai:Wso2ModelProvider` (auto-filled)

![Model Provider filled](/img/genai/tutorials/hr-knowledge-base-rag/25-model-provider-filled.png)

Click **Save**. The new connection appears in the **Connections** tree.

### 3.6 Generate the Answer

Click **+** below `ai:augmentUserQuery`. The **Model Providers** panel opens and lists `aiWso2modelprovider` with two actions — **Chat** and **Generate**. Hover **Generate** — the description reads *"Sends a chat request to the model and generates a value that belongs to the type corresponding to the type descriptor argument."* Click it.

![Generate action just before click](/img/genai/tutorials/hr-knowledge-base-rag/25a-generate-action-picker.png)

The **generate** form opens. Fill in:

- **Prompt** (Expression mode) —
  ```
  check aiChatusermessage.content.ensureType()
  ```
  The `content` field on `ai:ChatUserMessage` is typed as `string|ai:Prompt` — `ai:augmentUserQuery` populates it with one or the other depending on the augmentation strategy. The `generate` node's **Prompt** expects a Ballerina template literal (`string`-compatible), so use `ensureType()` to assert the `string` branch at runtime; `check` propagates any conversion error to the resource's error handler.
- **Result** — `result`
- **Expected Type** — `string`

![ai:generate filled](/img/genai/tutorials/hr-knowledge-base-rag/26-ai-generate-filled.png)

Click **Save**.

### 3.7 Return the Answer

Add a **Return** node (under **Control**) and return `result`. The completed query flow is:

```
Start → ai:retrieve (queryMatch)
      → ai:augmentUserQuery (aiChatusermessage)
      → ai:generate (result)
      → Return result
      → Error Handler
```

![Completed query flow](/img/genai/tutorials/hr-knowledge-base-rag/27-query-flow-complete.png)

## Step 4: Run and Try It

1. Set the `path` configurable to the folder that holds your HR documents (BI prompts you for this on first run; you can also edit `Config.toml` directly).
2. Run the **Automation** once so the in-memory store is populated with your HR documents — wait for the `Ingestion Completed!` log line before continuing.
3. Click **▶ Run** on the integration to start the HTTP Service.
4. Use the built-in **Try It** panel (top-right of the resource view) or `curl`. Tailor the question to a topic that appears in the documents you ingested:

   ```bash
   curl -X POST http://localhost:8080/api/v1/query \
     -H "Content-Type: application/json" \
     -d '{"userQuery": "What is the leave policy for new joiners?"}'
   ```

The response is the LLM's answer grounded in the chunks retrieved from your HR knowledge base. If the answer comes back as *"I don't have that information"*, double-check that the Automation finished ingesting and that the question matches a topic actually present in the source documents.

## What You Built

| Component | Where | Purpose |
|---|---|---|
| `path` (Configurable) | Configurations | Folder to ingest |
| `textDocumentLoader` (Data Loader) | Automation | Reads HR files from `path` |
| `aiInmemoryvectorstore` (Vector Store) | Connections | Stores embeddings |
| `aiWso2embeddingprovider` (Embedding Model) | Connections | Generates vector representations |
| `aiVectorknowledgebase` (Vector Knowledge Base) | Connections | Combines store + embedder + chunker |
| `aiWso2modelprovider` (Model Provider) | Connections | Calls the LLM |
| `ai:load` / `ai:ingest` | Automation | Loads, chunks, embeds, and writes documents |
| `ai:retrieve` | HTTP Service | Top-K vector search |
| `ai:augmentUserQuery` | HTTP Service | Builds the grounded chat message |
| `ai:generate` | HTTP Service | Generates the typed answer |

You now have a fully visual HR RAG pipeline that grounds an LLM in your actual policies — no glue code, and every connection is reusable across other artifacts in the same project.

## What's Next

- [AI Connections and Stores → Vector Stores](/docs/genai/develop/components/vector-stores) — Swap the in-memory store for a persistent backend (Pinecone, Milvus, Pgvector, Weaviate).
- [AI Connections and Stores → Chunkers](/docs/genai/develop/components/chunkers) — Tune chunk size and overlap, or plug in a custom chunker.
- [RAG → The Query Flow](/docs/genai/develop/rag/overview#the-query-flow) — Customize retrieval (top-K, filters, hybrid search).
- [AI Customer Support Agent](ai-customer-support.md) — Reuse `aiVectorknowledgebase` from a chat agent as a tool.
- [Multi-Agent Workflow](multi-agent-workflow.md) — Combine RAG with other agents in a larger workflow.
