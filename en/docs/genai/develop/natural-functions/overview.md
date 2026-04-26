---
sidebar_position: 1
title: Natural Functions
description: Reference for natural functions in WSO2 Integrator — typed Ballerina functions whose body is plain English, evaluated by an LLM at runtime.
---

# Natural Functions

A **natural function** is a Ballerina function whose body is written in **English** instead of code. You declare the signature — typed parameters, typed return — and you describe the work in a `natural { ... }` block. At runtime the LLM evaluates the description against the inputs and produces a value of the declared return type.

This section is a feature reference. For an end-to-end walkthrough see the [Natural Functions tutorial](/docs/genai/tutorials).

:::caution Experimental Feature
Natural expressions are an **experimental** language feature. The integration runtime adds the `--experimental` flag automatically when you click **Run** in BI; if you `bal run` from the terminal, pass `--experimental` yourself.
:::

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [The `natural { }` block](the-natural-block.md) | The English body of a natural function. | The body of any function flagged as a natural function in the Function editor. |
| [Typed return inference](typed-return-inference.md) | The return type drives the JSON schema sent to the LLM and the parsing of its response. | The **Return Type** field on the function. |
| [Calling from a flow](calling-from-a-flow.md) | Use a natural function in a resource, automation, or another function. | **Add Node** → **AI** → **Call Natural Function**. |

## When to Use a Natural Function

| Use a natural function when… | Look elsewhere when… |
|---|---|
| You have a single-step text task: classify, summarise, extract, rewrite. | The task needs tool calls or multi-step reasoning — use an [AI Agent](/docs/genai/develop/agents/overview). |
| You want the same prompt usable from many flows, fully typed. | You only need a one-off LLM call inside one flow — use a [`generate` node](/docs/genai/develop/direct-llm/the-generate-node) directly. |
| You want a function you can mock and unit-test. | The task needs to remember earlier turns of a conversation — use an agent with [memory](/docs/genai/develop/agents/memory). |
| You can describe the work in two or three sentences. | The work needs to look up data from your systems — combine with a tool (in an agent), or with [RAG](/docs/genai/develop/rag/overview). |

## How a Natural Function Looks

A finished natural function has three parts: a typed signature, a `natural { ... }` body, and a `return result;` line.

```ballerina
function summarizeReviews(string[] reviews) returns Summary|error {
    Summary|error result = natural {
        Analyze the customer reviews. Return a summary, an overall sentiment,
        and the top positive and negative themes.

        Reviews: ${reviews}
    };
    return result;
}
```

What you write yourself:

- The function signature (`function summarizeReviews(string[] reviews) returns Summary|error`).
- The English instructions inside `natural { ... }`.
- The interpolations of the parameters with `${...}`.

What WSO2 Integrator handles for you:

- Generating a JSON schema from the `Summary` return type.
- Sending the prompt and the schema to the project's default model provider.
- Parsing the response back into a typed `Summary` value.
- Surfacing any parsing errors as a Ballerina `error`.

## Where Natural Functions Live in BI

After definition, a natural function appears under the **Functions** category of the project sidebar with a small AI badge. It can be called from any flow exactly like a normal function — the **Add Node** panel exposes it under **AI → Call Natural Function**.

## Setup

Natural functions use the project's **default model provider for natural functions**. From the Command Palette run:

```
Ballerina: Configure default model for natural functions (Experimental)
```

Sign in with your WSO2 account when prompted. BI writes the configuration value into your project automatically. This is a one-time setup per project.

The first time you run an integration that uses a natural function, BI may also prompt for the `wso2aiKey` value. The **Configure Application** dialog handles this for you.

## What's Next

- **[The `natural { }` block](the-natural-block.md)** — the English body of the function.
- **[Typed Return Inference](typed-return-inference.md)** — how the return type drives output structure.
- **[Calling from a Flow](calling-from-a-flow.md)** — use a natural function from a resource or automation.
- **[What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function)** — conceptual background.
