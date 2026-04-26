---
sidebar_position: 2
title: The natural { } Block
description: Reference for the natural expression block — the English body of a natural function.
---

# The `natural { }` Block

The body of a natural function is a single Ballerina expression: `natural { ... }`. Inside the braces you write the task in English, with parameter interpolation. At runtime the runtime ships the text to the configured model provider, parses the response, and produces a typed value.

## Structure

```ballerina
ResultType result = natural {
    <Plain English description of what to do.>
    <Optionally: rules, format hints, examples.>

    <Inputs interpolated with ${parameter} >
};
```

There are no keywords, no DSL, no special syntax inside the braces — it really is plain text plus `${...}` interpolations. The whole block is treated as a Ballerina template literal.

## Selecting the Model

The block uses the project's **default model provider for natural functions**. If you need a different model for a specific function, name the provider explicitly:

```ballerina
final ai:ModelProvider analyst = check ai:getDefaultModelProvider();

ResultType result = natural (analyst) {
    ...
};
```

The optional `(analyst)` after `natural` overrides the default for that block only.

## Interpolation

Any in-scope value can be embedded with `${...}`:

| What you interpolate | What the model sees |
|---|---|
| `${reviewText}` | The string itself, inline. |
| `${reviews}` (an array) | Each item as JSON, joined into a list. |
| `${customer}` (a record) | The record serialised as JSON, so the model can read every field. |
| `${customer.name}` | Just the field. |
| `${reviews.length()}` | The result of any expression — function calls, field access, arithmetic. |

You don't need to escape values or build prompt strings yourself. The interpolation handles formatting and quoting.

## Anatomy of a Good `natural { }` Body

Three things tend to make natural-function bodies more reliable:

### 1. State the task in one sentence at the top

The first sentence is what the model latches onto. Keep it specific.

> *"Classify the following customer review as positive, negative, or neutral and explain why."*

Not:

> *"Take a look at this and tell me what you think."*

### 2. Add rules as a short bullet list

Models follow bullets better than long paragraphs.

> *Rules:*
> *- Use British English spelling.*
> *- Keep the explanation under 30 words.*
> *- Do not mention competitor names.*

### 3. Put the inputs at the end, after a clear divider

Inputs are easier to spot when they're separated from instructions:

```ballerina
SentimentResult result = natural {
    Classify the sentiment of the following review as positive, negative, or neutral.
    Provide a short explanation.

    Review: ${review}
};
```

## What *Not* to Put in the Block

- **The output schema.** The return type drives that. (See [Typed Return Inference](typed-return-inference.md).)
- **"Return JSON" instructions.** Same reason — the schema does this for you.
- **Large pieces of context unrelated to the task.** Tokens are paid; trim what you don't need.
- **Secrets.** Anything in a natural block is sent to the LLM provider on every call.

## Multiple Lines, Indentation, Markdown

Multi-line blocks are fine. Whitespace is preserved as-is in the prompt:

```ballerina
function classifyTicket(string ticket) returns Triage|error {
    Triage|error result = natural {
        Classify the support ticket below.

        Categories:
        - billing
        - technical
        - account
        - other

        Severity scale:
        1 = informational, 5 = critical.

        Ticket:
        ${ticket}
    };
    return result;
}
```

You can use Markdown inside the block — `# headings`, `**bold**`, fenced code. The model treats it as plain text, but the structure helps it parse the instructions.

## Errors

The expression yields `T|error`. The error variant covers:

- The model could not produce a value matching the return type after retries.
- The HTTP call to the model provider failed.
- The configured model provider is missing or misconfigured.

Bind to `T|error` and `check` (or handle) the error explicitly:

```ballerina
Summary|error result = natural { ... };
if result is error {
    log:printError("Natural function failed", 'error = result);
    return;
}
```

## Composing Natural Functions

A natural function is just a Ballerina function. You can:

- Call it from any flow (the BI Flow Designer surfaces them under **AI → Call Natural Function**).
- Call it from another natural function. (Useful for two-step prompts: extract first, then transform.)
- Pass its result into a tool, an agent, an HTTP service, or an automation.

There's nothing AI-specific about the rest of the integration. You get to keep all the regular language tools — types, generics, query expressions, error handling — and only the body of this one function is delegated to an LLM.

## What's Next

- **[Typed Return Inference](typed-return-inference.md)** — how the function's return type drives the prompt's output shape.
- **[Calling from a Flow](calling-from-a-flow.md)** — use the natural function from a resource or automation.
- **[Writing the Prompt](/docs/genai/develop/direct-llm/overview#writing-the-prompt)** — extra prompt-writing tips that apply here too.
