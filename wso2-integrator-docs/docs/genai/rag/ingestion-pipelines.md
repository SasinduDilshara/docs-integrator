---
sidebar_position: 3
title: "Build Document Ingestion Pipelines"
description: "Create ingestion integrations that load, chunk, embed, and store documents in a vector database."
---

# Build Document Ingestion Pipelines

An ingestion pipeline takes your raw documents and transforms them into searchable vector embeddings stored in a vector database. You build this pipeline once, then run it whenever your source content changes to keep your knowledge base current.

This guide walks you through building a complete ingestion integration in WSO2 Integrator, from loading documents to storing embeddings.

## Build an Ingestion Pipeline in the Visual Designer

This walkthrough creates a complete ingestion flow using the low-code visual designer. You'll load a markdown file, chunk the content, generate embeddings, and store them in a vector knowledge base.

:::note
This tutorial uses an in-memory vector store for simplicity. For production, use external vector stores like Pinecone, Milvus, or Weaviate. An in-memory vector store holds data in RAM — fast and easy to set up, but data is erased when you stop the integration.
:::

### Step 1: Create a new integration project

1. Click on the **BI** icon in the sidebar.
2. Click on the **Create New Integration** button.
3. Enter the project name as `rag_ingestion`.
4. Select a directory location by clicking on the **Select Path** button.
5. Click **Create New Integration** to generate the project.

![Create a new integration project](/img/genai/rag/ingestion/create-a-new-integration-project.gif)

### Step 2: Create an automation

An automation runs automatically when the integration starts. This ensures your data is loaded and ingested as soon as the application is running.

1. In the design screen, click on **+ Add Artifact**.
2. Select **Automation** under the **Automation** artifact category.
3. Click **Create** to open the flow editor.

![Create an automation](/img/genai/rag/ingestion/create-an-automation.gif)

### Step 3: Create a text data loader

1. Hover over the flow line and click the **+** icon to open the side panel.
2. Click on **Data Loader** from the **AI** section.
3. Click **+ Add Data Loader** to create a new instance.
4. Choose **Text Data Loader**.
5. Under the **paths** field, click on **+ Add Another Value** and add the path to your markdown file.
6. Set **Data Loader Name** as `loader`.
7. Click **Save** to continue.

![Create a data loader](/img/genai/rag/ingestion/create-a-data-loader.gif)

### Step 4: Load data using the data loader

1. In the **Data Loaders** section, click on `loader`.
2. Click on **load** to open the configuration panel.
3. Name the result as `doc`.
4. Click **Save** to complete the data loading step.

![Load data using the data loader](/img/genai/rag/ingestion/load-from-the-data-loader.gif)

This step wraps the file content into an `ai:Document` record, preparing it for chunking and embedding.

:::note
In WSO2 Integrator, an `ai:Document` is a generic container that wraps the content of any data source — such as a file, webpage, or database entry. It holds the main content plus optional metadata, which becomes useful during retrieval operations.
:::

### Step 5: Create a vector knowledge base

A vector knowledge base manages the ingestion and retrieval of documents through a vector store.

1. Hover over the flow line and click the **+** icon.
2. Select **Vector Knowledge Bases** under the **AI** section.
3. Click **+ Add Vector Knowledge Base** to create a new instance.
4. In the **Vector Store** section, click **+ Create New Vector Store** and choose **InMemory Vector Store**, then click **Save**.
5. In the **Embedding Model** section, click **+ Create New Embedding Model**, select **Default Embedding Provider (WSO2)**, then click **Save**.
6. For the **Chunker** setting, leave it at the default value of **AUTO** or create a new chunker.
7. Set the **Vector Knowledge Base Name** to `knowledgeBase`.
8. Click **Save** to complete the configuration.

![Create a vector knowledge base](/img/genai/rag/ingestion/create-a-vector-knowledge-base.gif)

:::warning Embedding Dimensions
The **Default Embedding Provider (WSO2)** generates dense vectors with **1536 dimensions**. If you're using an external vector store (Pinecone, Milvus, Weaviate, etc.), ensure your vector store index is configured to support 1536-dimensional vectors.
:::

### Step 6: Ingest data into the knowledge base

1. In the **Vector Knowledge Bases** section, click on `knowledgeBase`.
2. Click on **ingest** to open the configuration panel.
3. Provide `doc` as the input for **Documents**.
4. Click **Save** to complete the ingestion step.

![Ingest data into the knowledge base](/img/genai/rag/ingestion/ingest-data-with-vector-knowlege-base.gif)

This step chunks the document and sends the chunks to the vector store, converting each chunk into an embedding and storing them for future retrieval.

### Step 7: Add a confirmation message

1. Hover over the flow line and click the **+** icon.
2. Select **Log Info** under the **Logging** section.
3. Enter `"Ingestion completed."` in the **Msg** field.
4. Click **Save**.

![Add a confirmation message](/img/genai/rag/ingestion/add-a-confirmation-message.gif)

### Step 8: Configure default WSO2 provider and run

1. As the workflow uses the `Default Embedding Provider (WSO2)`, configure its settings:
   - Press `Ctrl/Cmd + Shift + P` to open the VS Code command palette.
   - Run the command: `Ballerina: Configure default WSO2 model provider`.
2. Click the **Run** button in the top-right corner to execute the integration.
3. Once the integration runs successfully, you will see the message `"Ingestion completed."` in the console.

![Configure default WSO2 provider and run the integration](/img/genai/rag/ingestion/run-the-integration.gif)

---

## Build an Ingestion Pipeline in Pro-Code

For more control and customization, you can build the pipeline entirely in Ballerina code.

## Data Flow Overview

```
Source Documents (PDF, HTML, Markdown)
        │
        ▼
   ┌──────────────┐
   │  Data Loader  │  ── Read files from disk, S3, or HTTP endpoints
   └──────┬───────┘
        ▼
   ┌──────────────┐
   │   Parser      │  ── Extract raw text from document formats
   └──────┬───────┘
        ▼
   ┌──────────────┐
   │   Chunker     │  ── Split text into segments with optional overlap
   └──────┬───────┘
        ▼
   ┌──────────────┐
   │  Embedder     │  ── Convert chunks to 1536-dimension vectors
   └──────┬───────┘
        ▼
   ┌──────────────┐
   │ Vector Store  │  ── Persist embeddings for retrieval
   └──────────────┘
```

## Step 1: Create the Project

1. Open VS Code and launch the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `rag_ingestion`.
4. Choose **Automation** as the project type — ingestion runs as a batch job, not a persistent service.

## Step 2: Configure Dependencies

Add the required dependencies to your `Ballerina.toml`:

```toml
[[dependency]]
org = "ballerinax"
name = "ai"
version = "1.0.0"

[[dependency]]
org = "ballerinax"
name = "openai"
version = "1.0.0"
```

## Step 3: Load Documents

Use a data loader to read source files. WSO2 Integrator supports loading from local directories, cloud storage, and HTTP endpoints:

```ballerina
import ballerinax/ai;
import ballerina/file;
import ballerina/io;
import ballerina/log;

configurable string docsPath = "./resources/docs";

// Load all Markdown files from a directory
function loadDocuments() returns ai:Document[]|error {
    ai:Document[] documents = [];
    file:MetaData[] files = check file:readDir(docsPath);

    foreach file:MetaData fileMeta in files {
        if fileMeta.absPath.endsWith(".md") {
            string content = check io:fileReadString(fileMeta.absPath);
            documents.push({
                content: content,
                metadata: {
                    "source": fileMeta.absPath,
                    "filename": check file:basename(fileMeta.absPath)
                }
            });
        }
    }
    log:printInfo("Loaded documents", count = documents.length());
    return documents;
}
```

:::tip
For production ingestion, load from cloud storage (S3, Azure Blob, GCS) using the appropriate [connector](/docs/connectors/cloud-services). This keeps your integration stateless and your content centralized.
:::

## Step 4: Parse and Clean Content

If you are working with PDFs or HTML, parse them into plain text before chunking:

```ballerina
function parseDocument(ai:Document doc) returns ai:Document {
    // Strip HTML tags if present
    string cleaned = re`<[^>]*>`.replaceAll(doc.content, "");
    // Normalize whitespace
    cleaned = re`\s+`.replaceAll(cleaned, " ").trim();
    return {content: cleaned, metadata: doc.metadata};
}
```

## Step 5: Chunk Documents

Split each document into smaller segments. Chunking strategy directly affects retrieval quality:

```ballerina
configurable int chunkSize = 500;
configurable int chunkOverlap = 50;

function chunkDocuments(ai:Document[] documents) returns ai:Chunk[] {
    ai:TextSplitter splitter = new ai:RecursiveCharacterSplitter({
        chunkSize: chunkSize,
        chunkOverlap: chunkOverlap,
        separators: ["\n\n", "\n", ". ", " "]
    });

    ai:Chunk[] allChunks = [];
    foreach ai:Document doc in documents {
        ai:Chunk[] chunks = splitter.split(doc);
        allChunks.push(...chunks);
    }
    log:printInfo("Created chunks", count = allChunks.length());
    return allChunks;
}
```

:::info
For detailed guidance on choosing chunk sizes and overlap, see [Choose Chunking & Embedding Strategies](chunking-embedding.md).
:::

## Step 6: Embed and Store

Convert chunks into vectors and store them in your vector database:

```ballerina
configurable string openaiApiKey = ?;

final ai:EmbeddingProvider embedder = check new ai:OpenAIEmbedder({
    apiKey: openaiApiKey,
    model: "text-embedding-ada-002"
});

final ai:VectorStore vectorStore = check new ai:PineconeStore({
    apiKey: pineconeApiKey,
    environment: pineconeEnv,
    index: pineconeIndex
});

function embedAndStore(ai:Chunk[] chunks) returns error? {
    // Process in batches to avoid rate limits
    int batchSize = 100;
    int total = chunks.length();

    foreach int i in 0 ..< total / batchSize + 1 {
        int startIdx = i * batchSize;
        int endIdx = int:min(startIdx + batchSize, total);
        ai:Chunk[] batch = chunks.slice(startIdx, endIdx);

        // Embed the batch
        float[][] embeddings = check embedder->embedBatch(
            batch.map(c => c.content)
        );

        // Store each chunk with its embedding
        foreach int j in 0 ..< batch.length() {
            check vectorStore->insert({
                content: batch[j].content,
                vector: embeddings[j],
                metadata: batch[j].metadata
            });
        }
        log:printInfo("Stored batch", batchNumber = i + 1);
    }
}
```

## Step 7: Wire It All Together

Combine the steps into a single automation entry point:

```ballerina
public function main() returns error? {
    log:printInfo("Starting RAG ingestion pipeline");

    // Load → Parse → Chunk → Embed → Store
    ai:Document[] docs = check loadDocuments();
    ai:Document[] parsed = docs.map(d => parseDocument(d));
    ai:Chunk[] chunks = chunkDocuments(parsed);
    check embedAndStore(chunks);

    log:printInfo("Ingestion complete",
        documentsProcessed = docs.length(),
        chunksStored = chunks.length()
    );
}
```

## Scheduling Incremental Updates

For content that changes regularly, schedule your ingestion pipeline to run on a timer or trigger it from a webhook:

```ballerina
import ballerina/task;

// Run ingestion every 6 hours
listener task:Listener ingestionScheduler = new ({
    intervalInMillis: 6 * 60 * 60 * 1000
});

service on ingestionScheduler {
    remote function onTrigger() returns error? {
        check runIngestion();
    }
}
```

:::warning
Incremental ingestion should track which documents have changed since the last run. Store a hash or timestamp per document to avoid re-processing unchanged content.
:::

For production deployment of your ingestion pipeline, see [Deploy & Operate](/docs/deploy-operate).

## What's Next

- [Choose Chunking & Embedding Strategies](chunking-embedding.md) — Optimize your chunking approach for better retrieval
- [Build a RAG Query Service](building-rag-service.md) — Create the query side that serves answers from your knowledge base
- [Connect to Vector Databases](vector-database.md) — Explore supported vector databases and configuration options
