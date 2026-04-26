---
sidebar_position: 5
title: Chunker
description: Reference for the chunker feature in WSO2 Integrator — automatic chunking, the Generic Recursive Chunker, disabling chunking, and custom chunks.
---

# Chunker

A **chunker** decides how a document gets split into the smaller pieces that go into the vector store. Chunking choice is one of the most consequential decisions in a RAG pipeline — it directly affects retrieval quality.

WSO2 Integrator gives you three options, in order of effort:

1. **Auto** — let the framework chunk for you (default).
2. **Generic Recursive Chunker** — a configurable chunker as a connection.
3. **Custom chunks** — disable chunking and pass `ai:Chunk[]` you built yourself.

## Where the Chunker Setting Lives

On the Vector Knowledge Base configuration form, the **Chunker** field defaults to `AUTO`. Click it to pick a different one.

## 1. AUTO (Default)

When **Chunker** is `AUTO`, the runtime picks a sensible chunking strategy based on the document type. For Markdown, paragraph boundaries; for HTML, structural elements; for PDF, page-and-section aware splits. You don't configure anything.

This is the right default. Use it unless you have a specific reason not to.

## 2. The Generic Recursive Chunker

For more control, add a **Generic Recursive Chunker** connection. Click **Chunker** → **+ Create New Chunker** to open the form:

![The Create Chunker side panel. Notice 'This operation has no required parameters. Optional settings can be configured below.' Advanced Configurations 'Expand' link. Chunker Name field set to aiGenericrecursivechunker. Result Type field set to ai:GenericRecursiveChunker. Save button.](/img/genai/develop/rag/04-create-chunker.png)

| Field | What it does |
|---|---|
| **Chunker Name** | Variable name in the generated source (`aiGenericrecursivechunker` by default). |
| **Result Type** | The Ballerina type — `ai:GenericRecursiveChunker`. |
| **Advanced Configurations** | Optional knobs — chunk size, overlap, separator preferences. |

The Generic Recursive Chunker splits text recursively along a list of separators (paragraphs first, then sentences, then characters), keeping pieces under a target size while preserving as much structure as possible. It's a reasonable default for arbitrary text where AUTO doesn't fit.

## 3. Custom Chunks (DISABLE)

When you want full control — for example, you've already chunked documents in another pipeline, or you want a domain-specific splitter — pass `ai:DISABLE` as the chunker. The Knowledge Base will ingest whatever `ai:Chunk[]` you hand it without any further splitting:

```ballerina
final ai:KnowledgeBase aiVectorKnowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider, ai:DISABLE);
```

Then your ingestion code produces `ai:Chunk[]`:

```ballerina
import ballerina/ai;
import ballerina/io;

string policyText = check io:fileReadString("./leave_policy.md");
string[] paragraphs = re `\n\n+`.split(policyText);

ai:Chunk[] chunks = from string paragraph in paragraphs
                    where paragraph.trim().length() > 0
                    select {content: paragraph.trim()};

check aiVectorKnowledgeBase.ingest(chunks);
```

The `ai:Chunk` record requires a `content` string. Anything else (metadata, source, page number) you add as fields the vector store will keep alongside the vector.

## A Quick Tour of Chunking Strategies

If you want to write your own splitter, these are the common strategies — each fits different content shapes.

| Strategy | Idea | Good for |
|---|---|---|
| **Fixed size** | Cut every N characters/tokens. | Uniform-length documents, code. |
| **Paragraph** | Cut at blank lines / `<p>` tags. | Markdown, plain prose. |
| **Sentence** | Cut at sentence boundaries. | Short documents, FAQs. |
| **Recursive** | Try paragraph → sentence → character, in that order, capping size. | Mixed-content documents. (The Generic Recursive Chunker.) |
| **Semantic** | Cut where topic shifts (using an embedding-based detector). | Long-form articles, transcripts. |
| **Section-aware** | Cut at heading boundaries. | Manuals, policies, technical docs. |

### Fixed-Size Example

```ballerina
function fixedSizeChunks(string text, int maxChars, int overlap) returns ai:Chunk[] {
    ai:Chunk[] chunks = [];
    int i = 0;
    int textLen = text.length();
    while i < textLen {
        int end = int:min(i + maxChars, textLen);
        chunks.push({content: text.substring(i, end)});
        i = end - overlap;
        if i < 0 { i = 0; }
    }
    return chunks;
}
```

### Contextual Chunking

A small but high-leverage trick: prepend document-level context to each chunk so chunks remain meaningful out of order.

```ballerina
function contextualChunks(string[] rawChunks, string title, string summary)
        returns ai:Chunk[] {
    return from string chunk in rawChunks
           select {
               content: string `Document: ${title}
Summary: ${summary}
---
${chunk}`
           };
}
```

The chunk's content now carries enough context that the LLM understands what it's looking at when retrieved in isolation. This often beats elaborate chunking strategies for very little code.

## Choosing a Chunking Approach

| Situation | Pick |
|---|---|
| You're getting started and don't have specific quality issues. | **AUTO**. |
| Retrieval quality is mixed and you suspect chunk boundaries. | **Generic Recursive Chunker**, then tune size if needed. |
| You already have well-formed chunks from another pipeline. | **DISABLE** + custom chunks. |
| You need very domain-specific splitting (legal, medical, code). | **DISABLE** + a hand-written splitter. |

A surprisingly robust default: `Generic Recursive Chunker` with chunk size around 500–800 tokens and ~10% overlap. Iterate from there.

## How to Tell Chunking Is Wrong

Symptoms that your chunker needs attention:

- **Retrieval returns the right document but the wrong section.** Chunks are too big.
- **Retrieval returns related but disconnected fragments.** Chunks are too small or split mid-thought.
- **The model frequently says "I don't know" even when the answer is in your docs.** Chunks lose context (no headings, no surrounding paragraph) — try contextual chunking.
- **Answers contradict each other depending on phrasing.** Two chunks contradict because they were sliced from related sections — try larger chunks or section-aware splitting.

## What's Next

- **[Knowledge Bases](knowledge-bases.md)** — where the chunker plugs in.
- **[Data Loaders](data-loaders.md)** — the documents that get fed to the chunker.
- **[Query Nodes](query-node.md)** — invoke `ingest` and `retrieve` from a flow.
