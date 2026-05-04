---
sidebar_position: 7
title: What is RAG?
description: A plain-English explanation of Retrieval-Augmented Generation — how to make an AI answer questions about your own documents.
---

# What is RAG?

**RAG** stands for **Retrieval-Augmented Generation**. In plain English, it's a way of letting an [LLM](what-is-llm.md) answer questions about *your* documents — files, manuals, policies, knowledge-base articles — even though the LLM was never trained on them.

If you have ever wished ChatGPT could answer questions using your company's HR handbook, your product documentation, or last quarter's reports, RAG is the technique that makes that possible.

## The Problem RAG Solves

An LLM is trained on a huge slice of public text. It does *not* know:

- Your company's internal policies.
- The contents of any document you have not shown it.
- Anything that happened after its training cutoff.

You could try fixing this by pasting all your documents into the prompt, but most companies have far more documentation than fits in any model's context window. RAG is the workaround: instead of giving the model **everything**, give it just **the few passages that are most likely to answer this specific question**.

## The Idea in One Picture

```bash
                    ┌──────────────────┐
   user's question  │                  │  the right
   ───────────────► │  Find the most   │ ─── few passages ───►  LLM  ───►  answer
                    │  relevant bits   │
                    │  of my documents │
                    └──────────────────┘
                              ▲
                              │
                       my big pile
                       of documents
```

Two halves: a **retrieval** half that searches your documents for whatever is relevant to the question, and a **generation** half that hands those passages to the LLM along with the question. The LLM produces a fluent answer, grounded in your actual documents instead of in whatever it happens to remember.

## Why Not Just Ctrl+F?

Plain keyword search is not enough. People ask questions in their own words, and the right document might not contain those exact words.

> *"How long do I get to settle in before I start losing PTO?"*

A keyword search for `"settle in"` and `"PTO"` will probably miss the policy that uses the phrase *"probationary period"* and *"paid time off"*.

RAG sidesteps this by searching on **meaning** instead of letters. It does this with two ideas:

| Concept | Plain-English meaning |
|---|---|
| **Embedding** | A way to turn a piece of text into a list of numbers (a vector) so that pieces with similar *meaning* end up near each other in number-space, even if they use different words. |
| **Vector store** | A database designed to find vectors that are close to each other very quickly. |

You convert every chunk of every document into a vector once, store them, and then at query time you convert the user's question into a vector too and look for the nearest matches. Result: *"settle in"* and *"probationary period"* end up near each other.

## How a RAG System Gets Built

A RAG pipeline has two phases. They happen at different times.

### Phase 1 — Ingestion (done in advance)

You take your documents and prepare them for fast search:

```bash
your documents  ─►  Chunking  ─►  Embedding  ─►  Vector Store
                    (split into     (turn each       (save the
                     smaller          chunk into      vectors so
                     passages)        a vector)       we can search them)
```

- **Chunking** — Documents are too big to embed whole, so you split them into pieces. Usually a paragraph or two each.
- **Embedding** — Each piece is turned into a vector by an *embedding model* (a small AI model that specialises in this). In WSO2 Integrator you use this through an **embedding provider** connection — see [Embedding Providers](/docs/genai/develop/components/embedding-providers).
- **Storing** — The vectors go into a vector store. From now on, you can ask *"give me the chunks closest to this query"* and it answers in milliseconds.

You only do this once per document. If a document changes, you re-ingest it.

### Phase 2 — Querying (every time the user asks something)

```bash
user's question  ─►  Embed the question  ─►  Vector search (top K matches)
                                                       │
                                                       ▼
                                          ┌─────────────────────────┐
                                          │ Send the question +     │
                                          │ those K chunks to LLM   │
                                          └─────────────────────────┘
                                                       │
                                                       ▼
                                                    answer
```

- **Embed the query** — same kind of vector, same embedding model as the documents.
- **Find the closest chunks** — typically the top 5 or 10. These are the passages most likely to contain the answer.
- **Generate** — bundle the chunks and the question together and ask the LLM to answer using them.

The result is a fluent answer that is grounded in your documents instead of in the model's training data.

## What RAG Buys You

| Without RAG | With RAG |
|---|---|
| Model only knows what was in its training data. | Model knows about anything you have ingested. |
| Updating a fact requires retraining or fine-tuning. | Updating a fact means re-ingesting the changed document. |
| Hallucination is hard to control — model just guesses. | You can ask the model to answer *only* from the retrieved chunks. |
| Hard to show *where* an answer came from. | Easy: each retrieved chunk has a source. You can cite it. |

That last point is often the biggest win for enterprise use cases. With RAG you can build an assistant that says *"Yes, you can carry over up to 5 days of leave (HR Handbook, section 4.3)"* — and you can link to the actual section, because the system knows which chunk it came from.

## RAG vs. Fine-Tuning

People often ask which is better. They solve different problems.

| Use RAG when… | Use fine-tuning when… |
|---|---|
| You want the model to know *facts* from your documents. | You want the model to *behave* differently — adopt a tone, follow a format, answer in a specific style. |
| Your data changes (new policies, new content). | The behaviour you want is stable and not data-dependent. |
| You want easy updates and citations. | You're comfortable retraining when things change. |

For most "let an LLM answer questions about my data" use cases, the answer is RAG.

## Where RAG Sits in WSO2 Integrator

RAG isn't a single feature — it's a pipeline made of several pieces, each of which is a feature in WSO2 Integrator:

- A **knowledge base** ties together a vector store and an embedding provider.
- A **vector store** holds the embeddings (in-memory for development, or a real vector database for production).
- An **embedding provider** generates the vectors (default WSO2-hosted, OpenAI, Azure, and others).
- A **chunker** decides how documents get split, with sensible defaults.
- A **data loader** reads documents in (PDF, Markdown, HTML, and more).
- A **query node** in your flow performs the retrieval and hands the chunks to the LLM.

You wire those together once, and from then on your integration can answer questions over your documents just like any other operation in your flow.

## What RAG Is Not

- **Not magic.** If a fact isn't in your documents, RAG won't conjure it up. It can only retrieve what you've ingested.
- **Not a replacement for clean data.** Bad chunks (whole books in one chunk, or single words split across chunks) make for bad answers. Chunking matters.
- **Not free.** Every query embeds the question, searches the vector store, then calls the LLM. There is real cost — though usually less than fine-tuning.

## What's Next

- [What is an LLM?](what-is-llm.md) — The model that produces the final answer.
- [What is an AI Agent?](what-is-ai-agent.md) — Agents can use RAG as one of their tools.
- [RAG](/docs/genai/develop/rag/overview) — Build a RAG pipeline in WSO2 Integrator.
