---
sidebar_position: 1
title: Direct LLM Calls
description: Reference for the Direct LLM features in WSO2 Integrator — model providers, the generate node, prompts, and typed responses.
---

# Direct LLM Calls

A **direct LLM call** is the simplest way to use AI in WSO2 Integrator: you add a node to a flow, write a prompt, and get a response. No agent loop, no memory, no tools — just one LLM round-trip in the middle of your integration.

This section is a **feature reference**. Each page covers one part of the Direct LLM surface in BI: where it lives in the UI, what it configures, and the Ballerina shape it generates. For an end-to-end walkthrough, see the [tutorials](/docs/genai/tutorials).

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [Model Providers](adding-a-model-provider.md) | The connection to the LLM (OpenAI, Anthropic, Azure, Mistral, Ollama, WSO2 Default, …). | **Connections** → **Add Connection**, or right side of any flow under **Model Providers**. |
| [The `generate` node](the-generate-node.md) | Sends one prompt to the LLM and binds the response to a variable. | **Add Node** → **AI** → expand a model provider → **Generate**. |
| [Prompts & interpolation](prompts-and-interpolation.md) | The text you send to the LLM, with optional `${var}` interpolation. | The rich-text editor inside any `generate` (or `chat`) node. |
| [Binding & typed responses](binding-and-typed-responses.md) | Map the LLM response to a Ballerina type — string, record, array — and let BI handle the JSON schema for you. | **Result** and **Expected Type** fields on the `generate` node. |

## When to Use Direct LLM Calls

| Use direct LLM when… | Look elsewhere when… |
|---|---|
| You need a single-shot LLM call inside a flow. | You want a typed function with an English body — use [Natural Functions](/docs/genai/develop/natural-functions/overview). |
| You want full control over the prompt at the point of use. | The same prompt is reused all over the codebase — wrap it in a [natural function](/docs/genai/develop/natural-functions/overview) instead. |
| The LLM doesn't need to call any tools or remember earlier turns. | The task needs tool calls, multi-step reasoning, or chat memory — use an [AI Agent](/docs/genai/develop/agents/overview). |
| You need to ground answers in your own documents. | Add [RAG](/docs/genai/develop/rag/overview) and pass the retrieved context into the prompt. |

## How a Direct LLM Call Looks in a Flow

After configuration, a typical resource flow that uses a direct LLM call looks like this:

```
Start  ─►  generate (model: emailGenerator)  ─►  Return
                       │
                       └── prompt: "..."
                       └── result:  generatedEmail
                       └── expected type: EmailGenerateResponse
```

Behind the scenes, the BI flow designer generates Ballerina code that uses the `ballerina/ai` module:

```ballerina
import ballerina/ai;

final ai:ModelProvider emailGenerator = check ai:getDefaultModelProvider();

EmailGenerateResponse generatedEmail = check emailGenerator->generate(
    `You are an email writing assistant. Write a short email...`
);
```

You write neither the import nor the call by hand — both are generated from the flow. But the underlying API is intentionally small (`->generate`, `->chat`) so that you can drop into source view, read it, and trust what's there.

## What's Next

- **[Adding a Model Provider](adding-a-model-provider.md)** — pick and configure the LLM.
- **[The `generate` Node](the-generate-node.md)** — add the node to a flow.
- **[Prompts & Interpolation](prompts-and-interpolation.md)** — write the prompt.
- **[Binding & Typed Responses](binding-and-typed-responses.md)** — get a typed value back.
