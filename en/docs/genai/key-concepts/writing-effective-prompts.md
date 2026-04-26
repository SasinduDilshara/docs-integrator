---
sidebar_position: 8
title: Writing Effective Prompts
description: Practices for writing prompts that work — interpolating variables, structuring long prompts, and what to leave out — applied across Direct LLM calls, Natural Functions, and Agent instructions.
---

# Writing Effective Prompts

A **prompt** is the instruction you send to an LLM. The same set of practices applies anywhere a prompt shows up in WSO2 Integrator BI:

- A `generate` node's **Prompt** field in a [Direct LLM call](/docs/genai/develop/direct-llm/overview).
- The body of a [Natural Function](/docs/genai/develop/natural-functions/overview).
- The **Instructions** field of an [AI Agent](/docs/genai/develop/agents/overview).

Models behave more predictably when prompts are short, structured, and free of conflicting signals. The page below collects the practices that have the biggest effect.

## Interpolation

Anywhere in a prompt you can refer to a value from the surrounding flow with `${variableName}`:

> *"Write a short email to* `${recipientName}` *confirming* `${meetingDate}`*. The sender's name is* `${senderName}`*."*

The prompt is stored as a Ballerina **template literal** (`` `...` ``). Picking a value from the BI prompt editor's **Insert** menu is the same as typing `${variableName}` by hand.

What you can interpolate:

| Kind | Example | What lands in the prompt |
|---|---|---|
| **Strings** | `${customerName}` | The string value, inline. |
| **Numbers, booleans, decimals** | `${amount}` | Their textual representation. |
| **Records** | `${customer}` | The record serialised as JSON, so the LLM can read every field. |
| **Arrays** | `${reviews}` | Each item serialised as JSON, joined into a list. |
| **Expressions** | `${reviews.length()}` | Any in-scope expression: function calls, field access, arithmetic. |

> **Tip:** Don't paste raw values that need quoting into the prompt — use interpolation. It avoids escaping issues and makes the template self-documenting.

## Structuring Long Prompts

For prompts longer than a paragraph, structure helps the model. Models follow bulleted rules and labelled sections more reliably than long paragraphs.

| Section header | Used for |
|---|---|
| **Role** | One sentence: *"You are a customer support assistant for ACME Inc."* |
| **Task** | What this specific call should produce. |
| **Constraints / Rules** | Bullet list of must-do and must-not-do. |
| **Format hints** | Style ("polite, concise"), length ("under 100 words"). |
| **Inputs** | The actual data, interpolated with `${...}`. |
| **Examples** | One or two exemplars when accuracy matters. |

If the task is unusual, give the model one or two worked examples right before the actual input:

> *"Classify each review as **positive**, **negative**, or **mixed**.*
>
> *Example:*
> *Review: "Great product but shipping took forever."*
> *Sentiment: mixed*
>
> *Now classify this one:*
> *Review:* `${review}` *"*

## What NOT to Put in the Prompt

| Don't include | Why |
|---|---|
| **The output schema.** *"Return JSON with fields title, topic, …"* | The runtime already drives this through the **Expected Type** — see [Typed Responses](/docs/genai/key-concepts/typed-responses). Hand-written schema is at best redundant, at worst it fails in production when the prompt and the type disagree. |
| **Secrets.** API keys, customer PII, anything you would not paste into a public conversation. | Anything in the prompt is sent to the LLM provider on every call. |
| **Megabytes of context.** | Prompts are paid per token. Trim, or use [RAG](/docs/genai/develop/rag/overview) to bring in only the relevant pieces. |
| **"Be smart" instructions.** *"Think carefully", "be very accurate"* | They don't help. Specific constraints do. |

## A Quick Checklist

Before you save a prompt, scan it once for:

1. **One clear task** — if you find yourself writing "and also …", consider splitting into two calls or two flows.
2. **No schema instructions** — set the Expected Type instead.
3. **No secrets** — substitute `configurable` references for any keys or PII.
4. **All inputs interpolated** — no copy-pasted values inside the template.
5. **Sections labelled** for any prompt over a paragraph.

A short, structured prompt with a clear Expected Type beats a long, prose-heavy prompt almost every time.

## What's Next

- **[Direct LLM Calls](/docs/genai/develop/direct-llm/overview)** — where the prompt editor lives in the BI canvas.
- **[Typed Responses](/docs/genai/key-concepts/typed-responses)** — what to do instead of putting a schema in the prompt.
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)** — package a reusable prompt as a typed function.
