---
sidebar_position: 4
title: Prompts & Interpolation
description: How to write the Prompt field in a generate or natural function node — structure, interpolation, and best practices.
---

# Prompts & Interpolation

A **prompt** is the instruction you send to the LLM. In WSO2 Integrator, prompts live inside `generate` nodes, `chat` nodes, [natural functions](/docs/genai/develop/natural-functions/overview), and AI Agent **Instructions**. The same rules apply to all of them.

## The Prompt Editor

Click any **Prompt** field — for example on a `generate` node — and BI opens a rich-text editor:

| Element | What it does |
|---|---|
| **Insert** | Insert a variable, an expression, or a code block. |
| **Bold / Italic / Link** | Standard text formatting (rendered as plain text in the prompt). |
| **H1 / Quote / Lists / Tables** | Structure the prompt visually. Helpful for long prompts. |
| **Magic-wand AI assist** | Suggests a prompt scaffold for you when you describe the task in one line. |
| **Preview / Source** | Toggle between the rendered preview and the raw template source. |

The prompt is stored as a Ballerina **template literal** (`` `...` ``). The editor is just a friendly view onto that template.

## Interpolation: Putting Variables into the Prompt

Anywhere in the prompt you can refer to a variable from the surrounding flow with `${variableName}`:

> *"Write a short email to **${recipientName}** confirming **${meetingDate}**. The sender's name is **${senderName}**."*

What you can interpolate:

| Kind | Example | What lands in the prompt |
|---|---|---|
| **Strings** | `${customerName}` | The string value, inline. |
| **Numbers, booleans, decimals** | `${amount}` | Their textual representation. |
| **Records** | `${customer}` | The record serialised as JSON, so the LLM can read every field. |
| **Arrays** | `${reviews}` | Each item serialised as JSON, joined into a list. |
| **Expressions** | `${reviews.length()}` | Any in-scope expression — function calls, field access, arithmetic. |

> **Tip:** Don't paste raw values that need quoting into the prompt; use interpolation. It avoids escaping issues and makes the template self-documenting.

## A Useful Prompt Skeleton

Most practical prompts have four parts:

```
You are <role>.

<context — what the task is, what the user is trying to do>

Rules:
- <constraint 1>
- <constraint 2>

Inputs:
${input1}
${input2}
```

Concrete example:

> *"You are an email writing assistant.*
>
> *Write a short email to **${recipientName}** asking for a 30-minute meeting next week to discuss **${intent}**. Suggest two time slots from the list below.*
>
> *Rules:*
> *- Keep the email under 150 words.*
> *- Use a polite, professional tone.*
> *- Do not mention internal product names.*
>
> *Time slots: **${timeSlots}**"*

This pattern works because it sets the role, scopes the task, lists the rules in a way the model can follow, and ends with the actual data.

## What *Not* to Put in the Prompt

- **The output schema.** The **Expected Type** field handles that. Asking for "JSON with fields X, Y, Z" in the prompt is redundant — and if the prompt says one thing and the type says another, the type wins. (See [Binding & Typed Responses](binding-and-typed-responses.md).)
- **Secrets.** API keys, customer PII, anything you would not paste into a public chat. Anything in the prompt is sent to the LLM provider on every call.
- **Megabytes of context.** Prompts are paid per token. Trim — or use [RAG](/docs/genai/develop/rag/overview) to bring in only the relevant pieces.
- **"Be smart" instructions.** *"Think carefully", "be very accurate"* don't help. Specific constraints do.

## Structuring Long Prompts

For prompts over a paragraph, structure helps the model:

| Section header | Used for |
|---|---|
| **Role** | One sentence: *"You are a customer support assistant for ACME Inc."* |
| **Task** | What this specific call should produce. |
| **Constraints / Rules** | Bullet list of must-do and must-not-do. |
| **Format hints** | Style ("polite, concise"), length ("under 100 words"). |
| **Inputs** | The actual data, interpolated with `${...}`. |
| **Examples** | One or two exemplars when accuracy matters. |

Models follow bulleted rules and labelled sections more reliably than long paragraphs. The Markdown the BI editor produces ends up as plain text in the template, but the structure helps the LLM all the same.

## Few-Shot Examples

If the task is unusual, give the model one or two worked examples right before the actual input:

> *"Classify each review as **positive**, **negative**, or **mixed**.*
>
> *Example:*
> *Review: "Great product but shipping took forever."*
> *Sentiment: mixed*
>
> *Now classify this one:*
> *Review: **${review}***"

For tasks with strong patterns this lifts accuracy more than rewriting the instructions five times.

## System Prompts vs. Instructions

The prompt inside a `generate` node is a **user message**. It is sent to the model along with any **system prompt** the provider has configured.

- For [AI Agents](/docs/genai/develop/agents/overview), the **Instructions** field on the agent is the system prompt — apply it once for the agent's whole life.
- For one-off `generate` calls, you can pass a per-call **System Message** under **Advanced Configurations**.

When in doubt, put role and rules in the system prompt and put data and the specific task in the user prompt. The model treats system prompts as more authoritative.

## Iteration: Improving a Prompt

Prompts are code. You iterate on them.

1. Run the flow with a representative input.
2. Look at where the response went wrong (missing field, wrong tone, drifted off topic).
3. Add the smallest possible rule to fix it.
4. Re-run, including with **adversarial** inputs (empty arrays, very long ones, edge cases).
5. Stop when the response is stable across the inputs you care about.

Don't grow prompts by accretion. If the rule list passes 10 bullets, consider whether the task should be split into two prompts, or whether a [natural function](/docs/genai/develop/natural-functions/overview) with a typed return would be cleaner.

## What's Next

- **[Binding & Typed Responses](binding-and-typed-responses.md)** — how Expected Type drives the prompt's output shape automatically.
- **[The `generate` Node](the-generate-node.md)** — back to the node configuration.
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)** — when the same prompt is used in many places, package it as a function.
