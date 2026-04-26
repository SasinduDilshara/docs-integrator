---
sidebar_position: 1
title: Direct LLM Calls
description: Single-page reference for Direct LLM calls in WSO2 Integrator, model providers, the generate node, prompts, and typed responses.
---

# Direct LLM Calls

A **direct LLM call** is the simplest way to use AI in WSO2 Integrator: you add a node to a flow, write a prompt, and get a typed response back. No agent loop, no memory, no tools. One round-trip.

This page is a single, end-to-end reference for everything you need to make that call work in BI: adding a model provider, dropping a `generate` node, writing the prompt, and binding the response to a Ballerina type.

> **Looking for a hands-on walkthrough?** See the **[Email Generator with Direct LLM](/docs/genai/tutorials/email-generator-direct-llm)** tutorial — it builds a `POST /emails/generate` service end to end using everything described on this page.

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

1. **Make sure a Model Provider connection exists** — the connection to the LLM. Adding one (and the per-provider form fields, model lists, and advanced HTTP knobs) is documented in **[AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers)**. You only need to do this once per project.
2. [**Add the Generate Node**](#the-generate-node), the call itself.
3. [**Write the Prompt**](#writing-the-prompt) and [**Pick the Expected Type**](#picking-the-expected-type).

The rest of this page walks each step in order.

> If your project does not have a model provider yet, head over to [AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers) first. The fastest one is the **Default WSO2 Model Provider** — no API key, just a one-time WSO2 sign-in.

---

## The Generate Node

The `generate` node is the workhorse of direct LLM calls. It sends a single prompt to a model provider and binds the response to a variable.

### Where to Find It

The `generate` action lives **on the model-provider connection itself**, not as a standalone node under AI. Once a provider exists, open the **Model Providers** panel on the right side of the flow editor, expand the connection (for example `azureOpenaimodelprovider`), and click **Generate**.

![The Model Providers right-side panel with azureOpenaimodelprovider expanded, showing the Generate action highlighted with a short description: it sends the prompt to the model and produces a value that conforms to the configured Expected Type. Other connections (anthropicModelprovider, openaiModelprovider, wso2ModelProvider) are collapsed below.](/img/genai/develop/direct-llm/15-generate-action-providers-panel.png)

### The Configuration Form

When the form opens, three fields are all you need: the **Prompt**, the **Result** variable, and the **Expected Type**. **Add the prompt that describes the work, pick the type you want the response in for your use case, and click Save.** Both fields are covered in detail in [Writing the Prompt](#writing-the-prompt) and [Picking the Expected Type](#picking-the-expected-type) below.

![The Generate configuration panel for openaiModelprovider → generate. Below the header: a Prompt* field with rich-text content; a Result* field set to 'result'; an Expected Type* field set to 'string'; a Save button.](/img/genai/develop/direct-llm/16-generate-form-rendered.png)

| Field | Required | What it does |
|---|---|---|
| **Prompt*** | Yes | The instruction sent to the LLM. Detailed in [Writing the Prompt](#writing-the-prompt). |
| **Result*** | Yes | The variable name where the response is stored. Used by later nodes. |
| **Expected Type*** | Yes | The Ballerina type the response should be parsed into. Detailed in [Picking the Expected Type](#picking-the-expected-type). |

There are no per-call overrides on the `generate` node — anything you'd tune (temperature, max tokens, etc.) lives on the model provider connection and applies to every call that uses it. See [AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers) for the full list of advanced configurations per provider.

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

### Prompt Practices

The same prompt-writing practices apply across `generate` nodes, [Natural Functions](/docs/genai/develop/natural-functions/overview), and AI Agent **Instructions**. They are documented once in the key concept page **[Writing Effective Prompts](/docs/genai/key-concepts/writing-effective-prompts)** — covering interpolation (`${variable}`), structuring long prompts with Role / Task / Constraints sections, and what to leave out (secrets, hand-written schemas, "be smart" instructions).

---

## Picking the Expected Type

The **Expected Type** field on the `generate` node is what makes the response come back as a real Ballerina value — a `string`, an `int`, a record, an array — not a blob of text you have to parse yourself.

The runtime derives a JSON schema from the type, asks the model to fill it, parses the response back, and hands the typed value to the next node. **You don't need to write any schema instructions in the prompt** — the type drives that automatically.

The full conceptual reference, including how to pick a type, why you should never describe the schema in the prompt, error handling, and tips for result types, is on the key concept page **[Typed Responses](/docs/genai/key-concepts/typed-responses)**.

A quick orientation:

| Use this Expected Type | When you want… |
|---|---|
| `string` | Free-form text — a summary, an email body, a translation. |
| A scalar (`int`, `decimal`, `boolean`) | A single number or yes/no answer. |
| A record | Named fields — the most common choice in integrations. |
| An array of records | A list of items. |
| A union (e.g. `Approved\|Rejected`) | The answer is one of several shapes; `match` on it. |

---

## Common Mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| Response comes back as a string of JSON instead of a parsed record. | Expected Type is `string`. | Set Expected Type to the record type. |
| Response stops half-way. | Hit the model's max output tokens. | Raise **Maximum Tokens** in the model provider's [Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations), or shorten the requested output. |
| Same prompt produces wildly different answers. | Temperature is high. | Lower the temperature on the model provider connection. |
| Parsing fails for some inputs in production. | The prompt and the type disagree, or the type is too loose. | Remove schema instructions from the prompt — see [Typed Responses → Don't put the schema in the prompt](/docs/genai/key-concepts/typed-responses#dont-put-the-schema-in-the-prompt); use closed records. |

## What's Next

- **[Email Generator with Direct LLM (Tutorial)](/docs/genai/tutorials/email-generator-direct-llm)** — build a complete `POST /emails/generate` service from scratch using everything on this page.
- **[Writing Effective Prompts](/docs/genai/key-concepts/writing-effective-prompts)** — interpolation, structuring long prompts, and what to leave out.
- **[Typed Responses](/docs/genai/key-concepts/typed-responses)** — the full reference for the Expected Type field.
- **[AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers)** — switch the LLM provider for production; tune temperature, max tokens, retries.
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)** — when the same prompt is used in many places, package it as a typed function.
- **[RAG](/docs/genai/develop/rag/overview)** — ground the model's answers in your own documents.
- **[AI Agents](/docs/genai/develop/agents/overview)** — if the task needs tools or multi-turn reasoning.
- **[What is an LLM?](/docs/genai/key-concepts/what-is-llm)** — conceptual background.
