---
sidebar_position: 3
title: The natural { } Block
description: Reference for the natural expression block — the English body of a natural function and the BI Prompt node that edits it.
---

# The `natural { }` Block

The body of a natural function is a single Ballerina expression: `natural { ... }`. Inside the braces you write the task in English, with parameter interpolation. At runtime the runtime ships the text to the bound model provider, parses the response, and produces a typed value.

In the BI Flow Designer the same block is shown as a single **Prompt** node sitting between **Start** and the end of the natural function flow.

![Empty Prompt node inside a natural function flow, with placeholder text "Enter your prompt here..." and a small cog icon to its right.](/img/genai/develop/natural-functions/18-natural-function-flow-empty.png)

| BI control | Source equivalent |
|---|---|
| **Prompt body** (the rich-text area) | The string contents of `natural { ... }`. |
| **Pencil icon** (top-right of the node) | Opens the Prompt editor for editing the body. |
| **Cog icon** (right of the node) | Opens **Configure Model Provider Connection**, equivalent to the optional `(provider)` argument on `natural`. |

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

The block uses the project's **default model provider for natural functions**. To override it, click the cog icon on the Prompt node and pick or create a different provider:

![Configure Model Provider Connection panel with Select Model Provider dropdown, "+ Create New Model Provider" link, a hint about the Configure default WSO2 model provider command, and a Save button.](/img/genai/develop/natural-functions/20-configure-model-provider-panel.png)

In source form this becomes the optional `(provider)` argument on the `natural` keyword:

```ballerina
final ai:ModelProvider analyst = check ai:getDefaultModelProvider();

ResultType result = natural (analyst) {
    ...
};
```

The optional `(analyst)` after `natural` overrides the default for that block only.

## Editing the Prompt in BI

Click the pencil icon on the Prompt node. An inline editor opens; click **Expand Editor** for the full editor with formatting tools.

![Prompt node with the pencil edit icon highlighted and a tooltip reading "Edit Prompt".](/img/genai/develop/natural-functions/22-prompt-edit-pencil.png)

The expanded editor exposes a Markdown toolbar:

![Expanded Prompt editor showing the toolbar with Insert, Undo/Redo, Bold, Italic, Underline, Link, H1, blockquote, lists, table, clear-formatting, plus Preview/Source toggle.](/img/genai/develop/natural-functions/23-prompt-rich-editor-empty.png)

| Toolbar action | Effect |
|---|---|
| **Insert** | Insert an `${parameter}` interpolation for any in-scope variable, or a snippet (block quote, code block). |
| **Bold / Italic / Underline** | Inline emphasis. The model treats Markdown emphasis as a hint to weight that term. |
| **H1**, **lists**, **blockquote**, **table** | Structural Markdown — useful for *Rules:*-style lists and *Examples:*-style sections. |
| **Preview / Source** | Toggle between rendered Markdown and the literal text the runtime sends to the model. |

A typical filled body:

![Expanded Prompt editor with the prompt typed: "You are a customer review analyzer. For each review, identify the overall sentiment, extract the key topics being discussed with their individual sentiment, and suggest a follow-up action if needed." The phrase "customer review analyzer" is bold.](/img/genai/develop/natural-functions/24-prompt-rich-editor-filled.png)

Click **Save**. The body collapses back into the Prompt node:

![Prompt node showing the saved prompt body inline.](/img/genai/develop/natural-functions/25-prompt-saved.png)

## Interpolation

Any in-scope value can be embedded with `${...}`. Use the **Insert** menu in the rich editor or just type the placeholder directly.

| What you interpolate | What the model sees |
|---|---|
| `${reviewText}` | The string itself, inline. |
| `${reviews}` (an array) | Each item as JSON, joined into a list. |
| `${customer}` (a record) | The record serialised as JSON, so the model can read every field. |
| `${customer.name}` | Just the field. |
| `${reviews.length()}` | The result of any expression — function calls, field access, arithmetic. |

You don't need to escape values or build prompt strings yourself. The interpolation handles formatting and quoting.

## Anatomy of a Good `natural { }` Body

Three things tend to make natural-function bodies more reliable.

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

You can use Markdown inside the block — `# headings`, `**bold**`, fenced code. The model treats it as plain text, but the structure helps it parse the instructions. The rich-text editor in BI produces exactly this Markdown source.

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
- **[Creating a Natural Function](creating-a-natural-function.md)** — the BI Create form, parameters, and return-type wizard.
