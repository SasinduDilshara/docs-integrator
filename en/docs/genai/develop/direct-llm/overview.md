---
sidebar_position: 1
title: Direct LLM Calls
description: Single-page reference for Direct LLM calls in WSO2 Integrator, model providers, the generate node, prompts, and typed responses.
---

# Direct LLM Calls

A **direct LLM call** is the simplest way to use AI in WSO2 Integrator: you add a node to a flow, write a prompt, and get a typed response back. No agent loop, no memory, no tools. One round-trip.

This page is a single, end-to-end reference for everything you need to make that call work in BI: adding a model provider, dropping a `generate` node, writing the prompt, and binding the response to a Ballerina type.

## When to Use Direct LLM Calls

| Use direct LLM when… | Look elsewhere when… |
|---|---|
| You need a single-shot LLM call inside a flow. | You want a typed function with an English body, use [Natural Functions](/docs/genai/develop/natural-functions/overview). |
| You want full control over the prompt at the point of use. | The same prompt is reused all over the codebase, wrap it in a [natural function](/docs/genai/develop/natural-functions/overview) instead. |
| The LLM doesn't need to call any tools or remember earlier turns. | The task needs tool calls, multi-step reasoning, or conversation history, use an [AI Agent](/docs/genai/develop/agents/overview). |
| You need to ground answers in your own documents. | Add [RAG](/docs/genai/develop/rag/overview) and pass the retrieved context into the prompt. |

## How a Direct LLM Call Looks in a Flow

A typical flow with a direct LLM call has a `generate` node sitting between your inputs and the next step, with a small wire to the model-provider connection on the right.

![A BI flow on the canvas showing Start → Declare Variable (blogTitle) → Declare Variable (blogContent) → openai:generate (result) connected to a small openaiModelprov node on the right with the OpenAI logo → log:printInfo with template `${result}` → Error Handler.](/img/genai/develop/direct-llm/18-generate-saved-flow.png)

To build this you do three things, in order:

1. **Make sure a Model Provider connection exists** — the connection to the LLM. Adding one (and the per-provider form fields, model lists, and advanced HTTP knobs) is documented in **[Components → Model Providers](/docs/genai/develop/components/model-providers)**. You only need to do this once per project.
2. [**Add the Generate Node**](#the-generate-node), the call itself.
3. [**Write the Prompt**](#writing-the-prompt) and [**Pick the Expected Type**](#binding-typed-responses).

The rest of this page walks each step in order.

> If your project does not have a model provider yet, head over to [Components → Model Providers](/docs/genai/develop/components/model-providers) first. The fastest one is the **Default WSO2 Model Provider** — no API key, just a one-time WSO2 sign-in.

---

## The Generate Node

The `generate` node is the workhorse of direct LLM calls. It sends a single prompt to a model provider and binds the response to a variable.

### Where to Find It

The `generate` action lives **on the model-provider connection itself**, not as a standalone node under AI. Once a provider exists, open the **Model Providers** panel on the right side of the flow editor, expand the connection (for example `azureOpenaimodelprovider`), and click **Generate**.

![The Model Providers right-side panel with azureOpenaimodelprovider expanded, showing the Generate action highlighted with a short description: it sends the prompt to the model and produces a value that conforms to the configured Expected Type. Other connections (anthropicModelprovider, openaiModelprovider, wso2ModelProvider) are collapsed below.](/img/genai/develop/direct-llm/15-generate-action-providers-panel.png)

### The Configuration Form

When the form opens, three fields are all you need: the **Prompt**, the **Result** variable, and the **Expected Type**. **Add the prompt that describes the work, pick the type you want the response in for your use case, and click Save.** Both fields are covered in detail in [Writing the Prompt](#writing-the-prompt) and [Binding Typed Responses](#binding-typed-responses) below.

![The Generate configuration panel for openaiModelprovider → generate. Below the header: a Prompt* field with rich-text content; a Result* field set to 'result'; an Expected Type* field set to 'string'; a Save button.](/img/genai/develop/direct-llm/16-generate-form-rendered.png)

| Field | Required | What it does |
|---|---|---|
| **Prompt*** | Yes | The instruction sent to the LLM. Detailed in [Writing the Prompt](#writing-the-prompt). |
| **Result*** | Yes | The variable name where the response is stored. Used by later nodes. |
| **Expected Type*** | Yes | The Ballerina type the response should be parsed into. Detailed in [Binding Typed Responses](#binding-typed-responses). |

There are no per-call overrides on the `generate` node — anything you'd tune (temperature, max tokens, etc.) lives on the model provider connection and applies to every call that uses it. See [Components → Model Providers](/docs/genai/develop/components/model-providers) for the full list of advanced configurations per provider.

### After Saving

Click **Save** and the node lands in the flow as `<provider>:generate`, with a small connection wire to its model-provider node on the right. From here you can chain follow-up nodes the same way you would after any other call. (See the canvas screenshot at the [top of this page](#how-a-direct-llm-call-looks-in-a-flow).)

---

## Writing the Prompt

The **Prompt** is the instruction you send to the LLM. The same rules apply across `generate` nodes, [natural functions](/docs/genai/develop/natural-functions/overview), and AI Agent **Instructions**.

### The Prompt Editor

Click any **Prompt** field and BI opens a rich-text editor in a dialog. The toolbar gives you the usual formatting tools (Insert, undo/redo, Bold, Italic, Link, headings, quote, lists, tables, magic-wand AI assist) and a **Preview / Source** toggle.

![The Prompt editor dialog open with the toolbar at the top and a structured prompt in the body: Role, Task, Output, Summary, and a Scores (1–10) table. The Preview tab is selected; no popups are open.](/img/genai/develop/direct-llm/20-prompt-editor-preview.png)

The **Insert** menu is the bridge between the prompt and the rest of your project. Open it to pull in values from anywhere in scope, request inputs, flow variables, configurables, project functions, or RAG documents, and they land in the prompt as `${...}` interpolations.

![The Prompt editor dialog with the Insert menu opened, showing five sub-options: Inputs, Variables (highlighted), Configurables, Functions, Documents, over the same Role / Task / Output / Scores body.](/img/genai/develop/direct-llm/17-prompt-editor-insert-menu.png)

| Element | What it does |
|---|---|
| **Insert** | Pull a value into the prompt: **Inputs** (resource path/query/payload params), **Variables** (anything in scope in the flow), **Configurables** (`Config.toml` values), **Functions** (calls into project functions), **Documents** (RAG retrieval results). |
| **Bold / Italic / Link** | Standard text formatting (rendered as plain text in the prompt). |
| **H1 / Quote / Lists / Tables** | Structure the prompt visually. Helpful for long prompts. |
| **Magic-wand AI assist** | Suggests a prompt scaffold for you when you describe the task in one line. |
| **Preview / Source** | Toggle between the rendered preview and the raw template source. |

The prompt is stored as a Ballerina **template literal** (`` `...` ``). The editor is just a friendly view onto that template; picking a value from the **Insert** menu is the same as typing `${variableName}` by hand.

### Interpolation

Anywhere in the prompt you can refer to a variable from the surrounding flow with `${variableName}`:

> *"Write a short email to* `${recipientName}` *confirming* `${meetingDate}`*. The sender's name is* `${senderName}`*."*

What you can interpolate:

| Kind | Example | What lands in the prompt |
|---|---|---|
| **Strings** | `${customerName}` | The string value, inline. |
| **Numbers, booleans, decimals** | `${amount}` | Their textual representation. |
| **Records** | `${customer}` | The record serialised as JSON, so the LLM can read every field. |
| **Arrays** | `${reviews}` | Each item serialised as JSON, joined into a list. |
| **Expressions** | `${reviews.length()}` | Any in-scope expression: function calls, field access, arithmetic. |

> **Tip:** Don't paste raw values that need quoting into the prompt; use interpolation. It avoids escaping issues and makes the template self-documenting.

### Structuring Long Prompts

For prompts over a paragraph, structure helps the model:

| Section header | Used for |
|---|---|
| **Role** | One sentence: *"You are a customer support assistant for ACME Inc."* |
| **Task** | What this specific call should produce. |
| **Constraints / Rules** | Bullet list of must-do and must-not-do. |
| **Format hints** | Style ("polite, concise"), length ("under 100 words"). |
| **Inputs** | The actual data, interpolated with `${...}`. |
| **Examples** | One or two exemplars when accuracy matters. |

Models follow bulleted rules and labelled sections more reliably than long paragraphs.

If the task is unusual, give the model one or two worked examples right before the actual input:

> *"Classify each review as **positive**, **negative**, or **mixed**.*
>
> *Example:*
> *Review: "Great product but shipping took forever."*
> *Sentiment: mixed*
>
> *Now classify this one:*
> *Review:* `${review}` *"*

### What NOT to Put in the Prompt

- **The output schema.** WSO2 Integrator handles this through the **Expected Type**, see [Binding Typed Responses](#binding-typed-responses) below. Asking for *"return JSON with fields title, topic, …"* is at best redundant, at worst it fails in production when the prompt and the type disagree.
- **Secrets.** API keys, customer PII, anything you would not paste into a public conversation. Anything in the prompt is sent to the LLM provider on every call.
- **Megabytes of context.** Prompts are paid per token. Trim, or use [RAG](/docs/genai/develop/rag/overview) to bring in only the relevant pieces.
- **"Be smart" instructions.** *"Think carefully", "be very accurate"* don't help. Specific constraints do.

---

## Binding Typed Responses

WSO2 Integrator supports **automatic type binding for Direct LLM calls**. You declare a Ballerina type as the **Expected Type** on the `generate` node and the runtime takes care of the rest: it derives a JSON schema from the type, instructs the LLM to fill it, parses the response back into a typed value, and hands it to the next node.

The point of type binding is to keep the result **accurate, user-friendly, and predictable**. The next node in your flow receives a real `string`, `int`, record, or array, not a blob of text you have to parse yourself.

### Pick a Type, Not a Prompt Instruction

You define the shape you want by picking a type. For a structured result, that type is usually a small project record. For example, a record called `BlogContent` with a `title` and a `topic` looks like this in the BI type editor:

![A BI type card titled BlogContent with two fields: title (string) and topic (string).](/img/genai/develop/direct-llm/19-blogcontent-type-card.png)

Set **Expected Type** to `BlogContent` on the `generate` node, and the result variable lands in the next node already typed: `blogContent.title` and `blogContent.topic` are real string fields.

| Use this Expected Type | When you want… |
|---|---|
| `string` | Free-form text: a summary, an email body, a translation. |
| A scalar (`int`, `decimal`, `boolean`) | A single number or yes/no answer. |
| A record (e.g. `BlogContent`) | Named fields, the most common choice in integrations. |
| An array of records (e.g. `Topic[]`) | A list of items: extracted entities, suggestions. |
| A union (e.g. `Approved\|Rejected`) | The answer is one of several shapes, `match` on it. |

### Don't Put the Schema in the Prompt

Because the type already drives the schema, **writing *"please return JSON with fields title, topic, …"* in the prompt is wrong**. It's redundant when it agrees with the type, and it fails in production when it doesn't:

- The type is enforced by the runtime; the prompt is a hint to the model. If they disagree, the type wins and the call may error on outputs the prompt encouraged.
- A change to the type silently invalidates the hand-written schema in the prompt.
- The prompt grows brittle and hard to read, mixing instructions for the user task with structural noise.

**Set the Expected Type, leave the schema out of the prompt.** The prompt should describe the *task*; the type drives the *shape*.

### When the LLM Doesn't Comply

The runtime returns `T|error`. If the model produces something the type can't accept, an error propagates up your flow, you can wrap with an error handler to retry or fall back. Strict (closed) records are easier to validate than open ones; the default WSO2 model and modern flagship models from each provider are reliable on schema adherence in practice.

### Tips for Result Types

- **Use plain field names that match how the task is described**, `summary` not `respText`.
- **Use enums (singleton union types) for fixed sets of values**, `"positive"|"negative"|"mixed"` rather than `string`.
- **Keep types small**, two or three fields beat fifteen.
- **Avoid nesting deeper than two levels**, it confuses the model and bloats the response.
- **Add field doc comments** (`# what this field is`). They're sent to the model alongside the schema and improve accuracy.

---

## Common Mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| Response comes back as a string of JSON instead of a parsed record. | Expected Type is `string`. | Set Expected Type to the record type. |
| Response stops half-way. | Hit the model's max output tokens. | Raise **Maximum Tokens** in the model provider's [Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations), or shorten the requested output. |
| Same prompt produces wildly different answers. | Temperature is high. | Lower the temperature on the model provider connection. |
| Parsing fails for some inputs in production. | The prompt and the type disagree, or the type is too loose. | [Pick a Type, Not a Prompt Instruction](#pick-a-type-not-a-prompt-instruction); use closed records. |

## What's Next

- **[Natural Functions](/docs/genai/develop/natural-functions/overview)**, when the same prompt is used in many places, package it as a typed function.
- **[RAG](/docs/genai/develop/rag/overview)**, ground the model's answers in your own documents.
- **[AI Agents](/docs/genai/develop/agents/overview)**, if the task needs tools or multi-turn reasoning.
- **[What is an LLM?](/docs/genai/key-concepts/what-is-llm)**, conceptual background.
