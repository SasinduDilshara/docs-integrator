---
sidebar_position: 1
title: What is an LLM?
description: A plain-English introduction to Large Language Models — the AI that powers chatbots like ChatGPT and the AI features in WSO2 Integrator.
---

# What is an LLM?

A **Large Language Model**, or **LLM**, is the kind of AI behind tools like ChatGPT, Claude, Gemini, and Copilot. You give it text — a question, an instruction, a document — and it gives you back text that looks like it was written by a human.

If you have ever asked ChatGPT to "write me an email to my landlord", "summarise this article", or "explain this error message", you have used an LLM. WSO2 Integrator lets you put that same kind of intelligence inside your own applications and integrations, instead of only using it through a chat window.

## How to Picture an LLM

Think of an LLM as a very well-read assistant who:

- Has read an enormous amount of text (books, websites, code, manuals, conversations).
- Has learned the *patterns* of language — how to summarise, classify, translate, explain, write, reason step-by-step.
- Does not actually "remember" specific facts the way a database does. It generates the most plausible next words, given everything it has read and what you just asked.

That last point matters: the LLM is a very fast, very fluent **pattern-completer**. It is not a search engine, and it does not have access to your data unless you give it that data in your message.

## The Two Things You Send and Get Back

When you use an LLM, you only need to think about two things:

| What | What it is | Example |
|---|---|---|
| **Prompt** | The text you send to the model. | *"Translate this email into Japanese: Hello, your order has shipped."* |
| **Response** | The text the model sends back. | *"こんにちは、ご注文は発送されました。"* |

Everything else — model size, training data, how it computes the answer — is the model provider's problem, not yours.

## Why a Single LLM Can Do So Many Things

Because an LLM is a general pattern-completer, the same model can:

- **Summarise** a long block of text into a few sentences.
- **Classify** a customer review as *positive*, *negative*, or *neutral*.
- **Extract** structured data ("pull the invoice number, date, and total from this email").
- **Translate** between languages.
- **Generate** drafts of emails, reports, marketing copy, code.
- **Explain** technical concepts in plain language.
- **Reason** step-by-step about a problem.

You don't write a different program for each task. You write a different *prompt*.

## A Few Words You'll See Often

You don't need to be an AI engineer to use LLMs in WSO2 Integrator, but a handful of words come up a lot. These are the only ones worth knowing up front.

| Term | Plain-English meaning |
|---|---|
| **Token** | The unit the model thinks in. Roughly 3–4 characters of English text per token. "Hello world" is about 2 tokens. Every model has a maximum number of tokens it can read and write in one go. |
| **Context window** | The model's "working memory" — the maximum amount of text it can look at and produce in a single call, measured in tokens. |
| **Prompt** | What you send to the model. |
| **System prompt** | A special prompt that sets the model's role and rules ("You are a polite customer-support assistant. Never reveal personal data."). It applies to the whole conversation. |
| **Temperature** | How creative or how predictable the model should be. Low temperature = consistent, focused answers. High temperature = more varied, more creative answers. For most integrations you want low. |
| **Hallucination** | When a model produces something that sounds correct but isn't. This is the single biggest gotcha with LLMs. |

## What LLMs Are Great At

- Anything where the input is **messy human text** and the output is also text.
- Tasks that are easy to *describe* but hard to *code* — "tell me whether this complaint is urgent", "rewrite this in a friendlier tone".
- Combining many small pieces of context into a coherent answer.

## What LLMs Are Not Great At

- **Up-to-date facts.** A model is trained on data up to a certain date and does not know what happened yesterday.
- **Doing things.** A raw LLM cannot click a button, send an email, or query your database on its own. It just produces text. (To make it act, you give it **tools** — see [What are Tools?](what-are-tools.md).)
- **Always being right.** Models can confidently produce incorrect answers, especially about specific facts ("hallucination").
- **Working with very large documents at once.** Anything you send must fit in the context window.

These limits aren't problems to fix — they are the reason WSO2 Integrator pairs LLMs with the rest of the toolbox: tools to take action, **RAG** to give the model your data, **memory** to keep context across turns, and **guardrails** to keep responses safe.

## Where LLMs Sit Inside WSO2 Integrator

Inside an integration, the LLM is one component you use through a **model provider** — the connection settings that tell your integration which LLM to talk to (OpenAI, Anthropic, Azure, Mistral, the WSO2 default provider, a local model, and so on). You configure this provider once, and every AI feature in your integration uses it:

- A simple **direct LLM call** sends one prompt and reads the response.
- A **natural function** lets you write part of your code in plain English; the LLM is what runs it.
- A **RAG pipeline** retrieves your documents and asks the LLM to answer using them.
- An **AI agent** lets the LLM call your tools to get things done.

The LLM is the brain. The rest of WSO2 Integrator is everything around the brain.

## What's Next

- [What is a Natural Function?](what-is-natural-function.md) — The simplest way to put an LLM inside your integration.
- [What is an AI Agent?](what-is-ai-agent.md) — When the LLM needs to take actions, not just answer questions.
- [What is RAG?](what-is-rag.md) — How to make an LLM answer questions about your own documents.
