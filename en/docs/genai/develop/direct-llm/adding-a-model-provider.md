---
sidebar_position: 2
title: Adding a Model Provider
description: Configure an LLM connection in WSO2 Integrator — Default WSO2, OpenAI, Anthropic, Azure OpenAI, Mistral, Ollama, and more.
---

# Adding a Model Provider

A **Model Provider** is the connection to an LLM. It is the first thing every AI feature in WSO2 Integrator depends on — direct LLM calls, natural functions, RAG, and AI agents all use a model provider to talk to the model.

This page is the reference for adding and configuring one in BI.

## Where Model Providers Live

A model provider is a **connection** in your project. Once you add it, it shows up in three places:

- The left **Connections** tree, under your project.
- The **Model Providers** panel on the right side of any flow editor.
- Whenever a node asks for a model — for example a `generate` node, a natural function, or the **Model** field of an agent.

You only need to add a provider once per project. After that every feature in the project can use it.

## Supported Providers

BI ships a built-in connector for each of the following providers. Pick the one whose **Module** matches the LLM you want to call.

| Provider | Module | Notes |
|---|---|---|
| **Default Model Provider (WSO2)** | `ballerina/ai` | Pre-configured, signed in with your WSO2 account. The fastest way to get started. No API key required. |
| **OpenAI** | `ballerinax/ai.openai` | GPT-4o, GPT-4, GPT-3.5. Requires an OpenAI API key. |
| **Anthropic** | `ballerinax/ai.anthropic` | Claude family. Requires an Anthropic API key. |
| **Azure OpenAI** | `ballerinax/ai.azure` | OpenAI models hosted on Azure. Requires Azure endpoint, deployment name, and key. |
| **Mistral** | `ballerinax/ai.mistral` | Open-weight Mistral and Mixtral models, EU-hosted. |
| **DeepSeek** | `ballerinax/ai.deepseek` | DeepSeek reasoning models. |
| **Google Vertex** | `ballerinax/ai.googlevertex` | Gemini models on Google Cloud Vertex AI. |
| **Ollama** | `ballerinax/ai.ollama` | Local model runner. No data leaves the machine — useful for development and on-prem. |
| **OpenRouter** | `ballerinax/ai.openrouter` | Routes to many models behind a single API. |

## Adding a Model Provider from a Flow

The fastest path is to add a provider as you build the flow:

1. Open any resource or function flow in the **Flow Designer**.
2. Click **+** between two nodes.
3. In the **Add Node** panel, scroll to **AI** and click **Model Provider**.
4. The **Select Model Provider** panel opens on the right with the full list.

![The Select Model Provider side panel listing all available model providers including Default WSO2, Anthropic, Azure OpenAI, DeepSeek, Google Vertex, Mistral, Ollama, OpenAI, and OpenRouter.](/img/genai/develop/agents/04-select-model-provider.png)

5. Click the provider you want.
6. Fill in the configuration form on the right (see below).
7. Click **Save**.

The new connection appears under **Connections** in the left tree and under **Model Providers** on the right of the flow editor.

## The Default WSO2 Model Provider

The Default Model Provider is the only one that does **not** require an API key. It is signed in with your WSO2 account and ready to use immediately — ideal for prototyping.

When you pick **Default Model Provider (WSO2)**, BI runs a one-time setup:

1. The Command Palette shows **Ballerina: Configure default WSO2 model provider**.
2. Sign in with your WSO2 account when prompted.
3. BI writes a `wso2aiKey` value into your project's `Config.toml` automatically.

You can also trigger this setup any time from the Command Palette:

| Command | What it does |
|---|---|
| `Ballerina: Configure default WSO2 model provider` | Sign in and set up the default provider for direct LLM calls. |
| `Ballerina: Configure default model for natural functions (Experimental)` | Same, but for [natural functions](/docs/genai/develop/natural-functions/overview). |

## Configuration Fields

Every provider's form has at least these fields:

| Field | What it controls |
|---|---|
| **Model Provider Name** | The variable name used in the generated code. Keep it descriptive — for example `emailGenerator` or `supportModel`. |
| **Result Type** | The Ballerina type of the connection. Defaults to the provider's own type (e.g., `openai:ModelProvider`). |
| **Advanced Configurations** | Expanded options: API key (or the `configurable` reference for it), model name (for example `gpt-4o`), temperature, top-p, timeouts. |

For external providers, BI prompts you to either paste the API key directly or reference a `configurable` value from `Config.toml`. **Always prefer the `configurable` reference** in production — it keeps secrets out of source control.

## What BI Generates

After you click **Save**, BI writes one line of Ballerina to your project:

**Default WSO2:**

```ballerina
final ai:ModelProvider emailGenerator = check ai:getDefaultModelProvider();
```

**OpenAI:**

```ballerina
configurable string openAiApiKey = ?;

final ai:ModelProvider emailGenerator = check new openai:ModelProvider(
    openAiApiKey, openai:GPT_4O
);
```

You can switch providers later — even from default to OpenAI to Azure — without changing any other code, because every provider satisfies the same `ai:ModelProvider` interface. Only the connection line changes.

## Using the Same Provider Across Features

Once a model provider exists in the project you can use it in:

- A `generate` node ([direct LLM call](the-generate-node.md)).
- A `chat` node (multi-turn message list).
- A [natural function](/docs/genai/develop/natural-functions/overview).
- The **Model** field of an [AI Agent](/docs/genai/develop/agents/overview).

The same `final ai:ModelProvider` is shared by all of them — there's no per-feature configuration to repeat.

## Tuning Model Behavior

Parameters that change *how* the model responds (temperature, top-p, presence penalty, max tokens) live on the model provider, not on the node that uses it. Edit the connection from the **Connections** tree and look under **Advanced Configurations**.

| Parameter | Plain-English effect |
|---|---|
| **Temperature** | Lower (0.0–0.3) = focused and deterministic. Higher (0.7–1.5) = creative and varied. Most integrations want low. |
| **Top-p** | Alternative to temperature; cap on which tokens the model is allowed to sample. Leave at the default unless you have a reason. |
| **Max output tokens** | Hard limit on how long the response can be. Useful when you know the answer should fit in a tweet, a JSON object, etc. |

## What's Next

- **[The `generate` Node](the-generate-node.md)** — use the provider in a flow.
- **[Prompts & Interpolation](prompts-and-interpolation.md)** — write the prompt that goes through the provider.
- **[What is an LLM?](/docs/genai/key-concepts/what-is-llm)** — conceptual background.
