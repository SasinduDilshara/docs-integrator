---
sidebar_position: 1
title: Natural Functions
description: Reference for natural functions in WSO2 Integrator BI — typed Ballerina functions whose body is plain English, evaluated by an LLM at runtime.
---

# Natural Functions

A **natural function** is a Ballerina function whose body is written in **English** instead of code. You declare the signature — typed parameters, typed return — and you describe the work in a `natural { ... }` block (the visual **Prompt** node in the BI editor). At runtime, WSO2 Integrator ships the description and the schema of the return type to the configured model provider, parses the response, and gives you back a typed value.

This section is the feature reference. For an end-to-end tutorial see [Review Summarizer with Natural Function](/docs/genai/tutorials/review-summarizer-natural-function). For the conceptual idea see [What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function).

:::caution Experimental Feature
Natural expressions are an **experimental** language feature. The integration runtime adds the `--experimental` flag automatically when you click **Run** in BI; if you `bal run` from the terminal, pass `--experimental` yourself.
:::

## Features at a Glance

| Page | What it covers | Where you find it in BI |
|---|---|---|
| [Creating a Natural Function](creating-a-natural-function.md) | The Create form, parameters, return type, type creation, prompt body. | Sidebar **Natural Functions** node + **+** button, or **Add Artifact → Other Artifacts → Natural Function**. |
| [The `natural { }` block](the-natural-block.md) | The English body of a natural function and how the **Prompt** node maps to it. | The Prompt node in the function flow — pencil icon to edit. |
| [Typed Return Inference](typed-return-inference.md) | How the return type drives the JSON schema sent to the LLM and the parsing of its response. | The **Return Type** field on the Create form and the typed result variable on the call site. |
| [Calling from a Flow](calling-from-a-flow.md) | Use a natural function from a resource, automation, or another function. | **Add Node** → **AI** → **Call Natural Function**. |

## When to Use a Natural Function

| Use a natural function when… | Look elsewhere when… |
|---|---|
| You have a single-step text task: classify, summarise, extract, rewrite. | The task needs tool calls or multi-step reasoning — use an [AI Agent](/docs/genai/develop/agents/overview). |
| You want the same prompt usable from many flows, fully typed. | You only need a one-off LLM call inside one flow — use a [`generate` node](/docs/genai/develop/direct-llm/overview#the-generate-node) directly. |
| You want a function you can mock and unit-test. | The task needs to remember earlier turns of a conversation — use an agent with [memory](/docs/genai/develop/agents/memory). |
| You can describe the work in two or three sentences. | The work needs to look up data from your systems, combine with a tool (in an agent), or with [RAG](/docs/genai/develop/rag/overview). |

## How a Natural Function Looks

A finished natural function has three parts: a typed signature, a `natural { ... }` body, and a `return result;` line.

```ballerina
function analyzeCustomerReviewes(string customerReview) returns ReviewResponse|error {
    ReviewResponse|error result = natural {
        You are a **customer review analyzer**. For each review, identify the
        overall sentiment, extract the key topics being discussed with their
        individual sentiment, and suggest a follow-up action if needed.

        Review: ${customerReview}
    };
    return result;
}
```

What you write yourself in the BI editor:

- The function signature, declared once in the **Create New Natural Function** form.
- The English instructions inside the **Prompt** node (the `natural { ... }` block).
- The interpolations of the parameters with `${...}`.

What WSO2 Integrator handles for you:

- Generating a JSON schema from the `ReviewResponse` return type.
- Sending the prompt and the schema to the bound model provider.
- Parsing the response back into a typed `ReviewResponse` value.
- Surfacing any parsing errors as a Ballerina `error`.

## Where Natural Functions Live in BI

After creation, a natural function appears under the **Natural Functions** category of the project sidebar. It can be called from any flow exactly like a normal function — the **Add Node** panel exposes it under **AI → Call Natural Function**. The model-provider connection BI created for the function shows up under **Connections** as `_<functionName>Model`.

![Project sidebar with the Natural Functions node hovered, showing the inline + button on the right.](/img/genai/develop/natural-functions/10-sidebar-natural-functions.png)

## Setup

Natural functions need two things in place before they will run:

### 1. A Default Model Provider

The Prompt node binds to a model provider connection. The fastest path is the WSO2-hosted default. From the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) run:

```
Ballerina: Configure default WSO2 model provider
```

![Command Palette open with the > Configure default model provider filter, showing matching commands "Ballerina: Configure default WSO2 Model Provider" and "Ballerina: Configure default model for natural functions (Experimental)".](/img/genai/develop/natural-functions/39-command-palette-natural-functions.png)

Sign in with your WSO2 account when prompted. BI writes the configuration values into the project automatically — this is a one-time setup per project.

If you'd rather use OpenAI, Anthropic, Azure OpenAI, or any other provider, click the cog icon on the Prompt node and use **+ Create New Model Provider** in the Configure Model Provider Connection panel. See [Creating a Natural Function → Configure the Model Provider](creating-a-natural-function.md#configure-the-model-provider) for the full flow.

![Model Providers picker listing Default Model Provider (WSO2), Anthropic, Azure OpenAI, Deepseek, Google Vertex, Mistral, Ollama, OpenAI.](/img/genai/develop/natural-functions/21-model-providers-list.png)

### 2. The `--experimental` Flag

Natural expressions are still experimental, so the runtime flag must be present. Clicking **Run** in BI sets it automatically; from the terminal use `bal run --experimental`.

The first time you run an integration that uses a natural function, BI may also prompt for `wso2aiKey` (or the corresponding key for your provider). The **Configure Application** dialog handles this for you.

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| `bal run` fails with *"natural expressions require --experimental"*. | The `--experimental` flag isn't on the command line. | Click **Run** in BI (it adds the flag), or run `bal run --experimental` in the terminal. |
| Compile error: *"Default model for natural functions not configured"*. | First-time setup missed. | Run **Ballerina: Configure default WSO2 model provider** from the Command Palette, or pick a provider via the cog icon on the Prompt node. |
| Returned record has empty / wrong fields. | Prompt body doesn't mention the fields the type expects. | Spell out each field by name in the body, or rely on the schema and add field descriptions on the type (see [Typed Return Inference](typed-return-inference.md)). |
| Natural function works in isolation but errors when called from a flow. | The bound parameter has a different shape than the function expects. | Hover the parameter in the Call Natural Function panel and bind to a value with the matching shape. |
| Same input gives different answers each run. | Default temperature isn't 0. | Lower the natural-function model provider's temperature in **Connections → Advanced Configurations**. |
| Result type is a union and parsing fails for one variant. | The schema wasn't tight enough — model produced something close but not exactly matching. | Use closed records and singleton-string unions (`"a"\|"b"\|"c"`); add field doc comments. |

## What's Next

- **[Creating a Natural Function](creating-a-natural-function.md)** — the BI walkthrough end to end.
- **[The `natural { }` block](the-natural-block.md)** — the English body of the function.
- **[Typed Return Inference](typed-return-inference.md)** — how the return type drives output structure.
- **[Calling from a Flow](calling-from-a-flow.md)** — use a natural function from a resource or automation.
- **[What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function)** — conceptual background.
