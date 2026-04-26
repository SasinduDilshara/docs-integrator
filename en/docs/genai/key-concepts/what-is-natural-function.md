---
sidebar_position: 2
title: What is a Natural Function?
description: A plain-English explanation of natural functions — code where the body is written in English instead of a programming language.
---

# What is a Natural Function?

A **natural function** is an integration feature where the body of a function — the part that says *what to do* — is written in **plain English** instead of code. You declare what goes in and what comes out, and you describe the work in a sentence or two. WSO2 Integrator hands the description to an LLM, and the LLM does the work.

It's the simplest way to use AI in an integration. You aren't building a chatbot. You aren't building an agent. You are just writing one function whose insides are powered by an LLM.

## The Idea in One Picture

A normal function:

```
input  ──►  [ a block of code I wrote ]  ──►  output
```

A natural function:

```
input  ──►  [ a sentence I wrote in English ]  ──►  output
                       (executed by an LLM)
```

You design the *signature* — the inputs the function takes and the shape of the output it returns. The body is just a description of the task, like:

> *"Read the customer review below. Decide whether it is positive, negative, or neutral. Pull out the top three things the customer liked and the top three things they disliked."*

The LLM reads that, looks at the actual review you passed in, and produces a result that fits the output shape you declared.

## When You Would Use One

Natural functions shine when:

- The task is **easy to describe but hard to code by hand**: "summarise this", "classify this", "extract these fields from this email".
- The task is a **single shot** — one input, one answer. There are no follow-up questions, no decisions about which tool to use next, no conversation to keep track of.
- You want a **typed, structured result**, not just a blob of text.

If you find yourself thinking "I could explain this to a smart intern in two sentences, but writing the regex / parser / rules engine to do it would take a week", that's a natural function.

| Good fit | Not a fit |
|---|---|
| Classify a support ticket as *bug*, *billing*, or *feature request* | Multi-turn conversation with a customer |
| Summarise a meeting transcript into bullet points | Anything that needs to call APIs or databases to take action |
| Extract `name`, `email`, `order_id` from a free-form email | Tasks with strict, deterministic rules ("if total > 100, apply 10% discount") |
| Rewrite a paragraph in a friendlier tone | Tasks where you need to remember earlier interactions |

For anything in the right column, you probably want an [AI agent](what-is-ai-agent.md) or a regular function.

## What "Typed Output" Means and Why It Matters

A normal LLM call returns a **string** — free-form text. That's fine for a chatbot but awkward inside an integration, where the next step usually expects a real value: a record, a number, a list.

With a natural function you declare the output type up front — for example, *"this returns a record with fields `sentiment`, `summary`, and `topThemes`"*. WSO2 Integrator turns that type into a schema for the LLM and asks it to produce data that fits the schema. You get back a real, typed value you can use directly in the rest of your flow — no JSON parsing, no string-massaging, no "did the LLM remember to include all the fields?" anxiety.

This is the single biggest reason natural functions feel different from raw "call the LLM and parse the answer" code: the type system does the parsing and the validation for you.

## What a Natural Function is *Not*

- **Not a chatbot.** Each call is a fresh transaction with no memory of earlier calls.
- **Not an agent.** It can't decide to look up a customer record, query a database, or send an email. It only produces an answer from the input you give it.
- **Not a rules engine.** Two calls with the same input may produce slightly different wording. The *shape* of the answer is locked in by the type, but the exact phrasing comes from a probabilistic model.

If you need any of those things, you are looking for a different feature:

- Something that **takes actions** → [AI agent](what-is-ai-agent.md) with [tools](what-are-tools.md).
- Something that **remembers earlier turns** → an agent with [memory](what-is-agent-memory.md).
- Something that **answers from your documents** → [RAG](what-is-rag.md).

## Where a Natural Function Lives in an Integration

Inside WSO2 Integrator a natural function is just like any other function. You can:

- Call it from inside an HTTP service to add intelligence to an endpoint.
- Call it from a scheduled automation that processes a batch of records.
- Reuse the same natural function from many flows.
- Test it like any other function (with the usual caveats about LLMs being non-deterministic).

You define it once, and the rest of your integration treats it as a normal building block — the AI part stays neatly tucked inside the function body.

## What's Next

- [What is an LLM?](what-is-llm.md) — The model that actually runs the function body.
- [What is an AI Agent?](what-is-ai-agent.md) — When you need more than a single-step transformation.
- [Natural Functions](/docs/genai/develop/natural-functions) — How to build one in WSO2 Integrator.
