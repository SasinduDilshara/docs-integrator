---
sidebar_position: 6
title: Data Loaders
description: Reference for data loaders in WSO2 Integrator — read Markdown, HTML, PDF, DOCX, and PPTX files into a RAG pipeline.
---

# Data Loaders

A **data loader** reads documents from disk (or any source) into the form a Knowledge Base can ingest. WSO2 Integrator's `ai:TextDataLoader` covers the common formats out of the box; for everything else you can produce `ai:Document` or `ai:Chunk` values yourself and pass them to `ingest`.

## The Data Loaders Panel

In the right-side panel of any flow editor, the **Data Loaders** category shows the loaders available in the project:

![The Data Loaders side panel showing the Text Data Loader option with description 'Dataloader that can be used to load supported file types as TextDocument's. Currently only…'.](/img/genai/develop/rag/05-data-loaders.png)

The Text Data Loader is the default. New loader types appear here as they're released; today, you typically pick **Text Data Loader** for file-based documents and write a small loader function for other sources.

## Text Data Loader

The Text Data Loader reads files into `ai:Document` values that retain the original metadata (filename, format, modification time). Supported formats:

| Extension | Notes |
|---|---|
| `.md` | Markdown — paragraphs preserved, headings retained as metadata. |
| `.html` | HTML — text content with structural metadata. |
| `.pdf` | PDF — page-by-page text extraction. |
| `.docx` | Microsoft Word documents. |
| `.pptx` | Microsoft PowerPoint slides. |

```ballerina
import ballerina/ai;

ai:DataLoader loader = check new ai:TextDataLoader("./policy.md");
ai:Document|ai:Document[] documents = check loader.load();
check knowledgeBase.ingest(documents);
```

You can pass:

- A single file path: `"./policy.md"`.
- A directory (loads every supported file in it).
- A list of file paths.

Whatever the loader returns can be fed straight into the Knowledge Base's `ingest` action. The chunker (configured on the Knowledge Base) splits each document into chunks; the embedding provider turns those into vectors; the vector store persists them.

## Loading from Sources Other Than Disk

`ai:DataLoader` is just an interface. For documents that come from elsewhere — an S3 bucket, a database, a CMS, a SaaS API — write a small function that produces `ai:Document` (or directly `ai:Chunk[]`) and pass the result to `ingest`. The Knowledge Base does not care where the documents originated.

```ballerina
function loadFromSalesforce() returns ai:Document[]|error {
    SalesforceArticle[] articles = check sfClient->/articles;
    return from SalesforceArticle a in articles
           select {
               content: a.body,
               metadata: {
                   "id": a.id,
                   "title": a.title,
                   "updated": a.lastModifiedDate
               }
           };
}

ai:Document[] articles = check loadFromSalesforce();
check knowledgeBase.ingest(articles);
```

If you've already pre-chunked content (as discussed in [Chunker](chunker.md#3-custom-chunks-disable)), pass an `ai:Chunk[]` directly — same `ingest` call, no further splitting.

## Building an Ingestion Flow

A typical ingestion flow looks like this:

```
Start ─► Text Data Loader  (load policy files)
      │
      ▼
   Knowledge Base.Ingest  (chunk, embed, store)
      │
      ▼
    Return  (count of ingested documents)
```

This is usually exposed as:

- A scheduled **automation** that re-ingests on a cron.
- An admin-only **HTTP endpoint** so docs can be re-ingested on demand.
- A startup hook in `main()` for development.

A common production pattern is a flag-controlled re-ingestion plus targeted **Delete By Filter** for documents that have changed.

## Performance Tips

- **Batch loads, batch ingests.** Loading 100 small files individually and ingesting one-by-one is much slower than loading them all at once into a `Document[]` and calling `ingest` once.
- **Tag your chunks.** Add metadata (source ID, version, last-modified date) at load time so you can filter and clean up later.
- **Watch the embedding bill.** Re-ingestion re-embeds everything. Use **Delete By Filter** plus targeted ingestion when only a subset has changed.

## What's Next

- **[Chunker](chunker.md)** — how loaded documents get split.
- **[Knowledge Bases](knowledge-bases.md)** — the ingest target.
- **[Query Nodes](query-node.md)** — once data is in, query against it.
