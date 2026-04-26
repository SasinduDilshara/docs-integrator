---
sidebar_position: 3
title: The generate Node
description: Reference for the generate node — single-shot LLM invocation inside a WSO2 Integrator flow.
---

# The `generate` Node

The `generate` node is the workhorse of [direct LLM calls](overview.md). It sends a single prompt to a model provider and binds the response to a variable that the rest of your flow can use.

There is also a sister node, **`chat`**, for multi-turn message lists. This page focuses on `generate`; `chat` is described at the bottom.

## Where to Find It

There are two paths to a `generate` node, and they produce the same configuration form.

**Path 1 — From the Add Node panel**

1. Click **+** between two nodes in any flow.
2. In **Add Node** scroll to **AI**.
3. Pick **Generate**, then choose the model provider when prompted.

**Path 2 — From the Model Providers panel** (faster when the provider already exists)

1. On the right side of the flow editor open the **Model Providers** panel.
2. Expand the connection you want to use (for example `emailGenerator`).
3. Click **Generate**.

Both paths open the same right-side configuration panel.

## The Configuration Form

| Field | Required | What it does |
|---|---|---|
| **Prompt** | Yes | The instruction sent to the LLM. Opens in a rich-text editor; supports `${variable}` interpolation against any in-scope value. |
| **Result** | Yes | The variable name where the response is stored. Used by later nodes. |
| **Expected Type** | Yes | The Ballerina type the response should be parsed into. `string` for plain text, or any record/array/scalar type. |
| **Advanced Configurations** | No | Per-call overrides — temperature, max tokens, system message override. Defaults come from the model provider. |

The defaults are sensible. **Prompt + Result + Expected Type** is enough for almost every use case.

## Picking the Right Expected Type

`Expected Type` is the most consequential field. It determines how the LLM response is parsed and what the next node sees.

| Expected type | Use when… | Behaviour |
|---|---|---|
| `string` | You just want raw text — a paragraph, an email body, a translation. | The response is the LLM's text output, unchanged. |
| A record type | You want a structured result with named fields (subject + body, sentiment + confidence + reasons, …). | BI infers a JSON schema from the type and instructs the LLM to fill it. The response is parsed into a typed value. |
| An array type | You want a list of items (extracted entities, suggested tags, …). | The LLM produces a JSON array; BI parses it into the array type. |
| A union type | The answer can be one of several shapes (e.g. *"either a `Refund` or a `RejectionReason`"*). | The LLM picks the matching variant and BI parses accordingly. |

**You do not write *"please return JSON"* in the prompt.** The Expected Type does that for you. The prompt should describe the *task*; the type drives the *shape*.

## What BI Generates

For a `string` result:

```ballerina
string generatedEmail = check emailGenerator->generate(
    `You are an email writing assistant. Write a short email...`
);
```

For a record result with `Expected Type = EmailGenerateResponse`:

```ballerina
EmailGenerateResponse generatedEmail = check emailGenerator->generate(
    `You are an email writing assistant. Write a short email...`
);
```

The only difference is the variable type — the call is the same. BI handles the JSON-schema work behind the scenes.

## After the Node

The variable named in **Result** is now in scope. Common follow-up nodes:

- **Return** — send the value as the HTTP response.
- **Update Variable** — feed it into a later step.
- **Map Data** — transform it before returning.
- **If / Match** — branch on a field of the response.

## The `chat` Node — Multi-turn Variant

When you need a back-and-forth conversation rather than a single shot, use the **Chat** action under the model provider. The configuration form replaces **Prompt** with a **Messages** field that takes a list of `ai:ChatMessage` values (system, user, assistant). Use it when:

- You are building your own chat surface and want full control of the message history.
- You need to send the model a system message different from the provider's default.
- You want to mix multiple user/assistant turns in one call.

For most cases — including when you want chat-like behaviour from an agent — pick an [AI Agent](/docs/genai/develop/agents/overview) instead. Agents wrap `chat` plus tools, memory, and a reasoning loop, so you do not have to build the conversation logic yourself.

## Common Mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| Response comes back as a string of JSON instead of a parsed record. | Expected Type is `string`. | Set Expected Type to the record type. |
| Fields in the parsed record are sometimes empty. | Prompt doesn't mention all the fields the type expects. | List every field by name in the prompt: *"return `subject`, `content`, …"*. |
| Response stops half-way. | Hit the model's max output tokens. | Raise **Advanced Configurations → Max Tokens** on the provider, or shorten the requested output. |
| Same prompt produces wildly different answers. | Temperature is high. | Lower the temperature on the model provider connection. |

## What's Next

- **[Prompts & Interpolation](prompts-and-interpolation.md)** — how to write the Prompt field.
- **[Binding & Typed Responses](binding-and-typed-responses.md)** — go deeper on Expected Type.
- **[Adding a Model Provider](adding-a-model-provider.md)** — if you don't have a provider yet.
