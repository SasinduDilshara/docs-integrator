---
sidebar_position: 5
title: Binding & Typed Responses
description: How the Expected Type field on a generate node turns LLM output into a typed Ballerina value.
---

# Binding & Typed Responses

Out of the box, an LLM returns text. WSO2 Integrator turns that text into a **typed Ballerina value** automatically. You declare the shape you want; BI handles the rest — JSON-schema generation, parsing, validation.

This page explains the two fields that drive that behaviour: **Result** and **Expected Type**.

## The Two Fields

Both fields appear on every `generate` node, every `chat` node, and (implicitly) on every [natural function](/docs/genai/develop/natural-functions/overview).

| Field | What it does |
|---|---|
| **Result** | The variable name where the parsed response is stored. |
| **Expected Type** | The Ballerina type the response should conform to. |

Setting these two right is what changes a "call the LLM and parse text yourself" pattern into something you can compose with the rest of your flow.

## Five Common Type Choices

### 1. `string`

For free-form text:

| Field | Value |
|---|---|
| Expected Type | `string` |
| Result | `summary` |

```ballerina
string summary = check model->generate(`Summarise this article: ${article}`);
```

The response is whatever text the LLM produced, with no parsing.

### 2. A scalar — `int`, `decimal`, `boolean`

For numeric or yes/no answers:

| Field | Value |
|---|---|
| Expected Type | `int` |
| Result | `priority` |

```ballerina
int priority = check model->generate(
    `Score this ticket's urgency from 1 (low) to 5 (critical): ${ticketBody}`
);
```

BI instructs the LLM to produce a value that fits the type, and parses it back. If the model replies with `"2"`, you get the integer `2`.

### 3. A record type

For structured data with named fields. This is the most common choice for direct LLM calls inside an integration.

```ballerina
type EmailGenerateResponse record {|
    string subject;
    string content;
|};
```

| Field | Value |
|---|---|
| Expected Type | `EmailGenerateResponse` |
| Result | `generatedEmail` |

```ballerina
EmailGenerateResponse generatedEmail = check model->generate(`Write an email...`);
```

`generatedEmail.subject` and `generatedEmail.content` are real string fields, ready to return as the HTTP response or feed into the next node.

### 4. An array of records

For lists — extracted entities, suggestions, line items.

```ballerina
type Topic record {|
    string label;
    decimal confidence;
|};
```

| Field | Value |
|---|---|
| Expected Type | `Topic[]` |
| Result | `topics` |

```ballerina
Topic[] topics = check model->generate(
    `Pull out the top 5 themes from these reviews: ${reviews}`
);
```

### 5. A union — *"one of these shapes"*

For decisions where the answer can be one of several variants:

```ballerina
type RefundApproved record {|
    "approved" decision;
    string reason;
    decimal amount;
|};

type RefundRejected record {|
    "rejected" decision;
    string reason;
|};

type RefundDecision RefundApproved|RefundRejected;
```

| Field | Value |
|---|---|
| Expected Type | `RefundDecision` |
| Result | `decision` |

The LLM picks the variant it thinks is right, fills the fields, and BI gives you a typed `decision`. You then `match` on it the way you would on any union.

## Why You Don't Write *"Return JSON"* in the Prompt

The Expected Type *is* the schema. BI generates a JSON schema from the Ballerina type and includes it as part of the request to the model. The model is instructed to return data matching that schema.

That means:

- You don't need to spell out the field names in the prompt — the schema already does.
- If the type changes (you add a field, rename a field), your prompt does **not** need to change.
- You don't write JSON parsing code. The result is already the right type.

If you do put JSON instructions in the prompt and they conflict with the type, the type usually wins because the schema is more constraining than the natural-language hint.

## What Happens When the LLM Doesn't Comply

Models very occasionally return malformed output (missing field, wrong type for a field, extra prose around the JSON). The runtime handles this in two ways:

- **`check`** — the call returns `T|error`. If parsing fails, an error propagates up the flow. Wrap with an error handler if you want to retry or fall back.
- **Stricter types** — a **closed record** (declared with the `record &#123;| ... |&#125;` syntax) is harder for the model to satisfy than an **open record** (`record { ... }`). For *"this and only this"*, prefer closed records.

You will see this most often with very small or very old models. The default WSO2 model and modern flagship models from each provider are reliable on schema adherence.

## Tips for Designing Result Types

| Tip | Why |
|---|---|
| **Use plain field names that match the prompt's wording.** `summary` not `respText`. | The model picks better fields when the names match the natural way the user might describe them. |
| **Use enums (singleton union types) for fixed sets of values.** `"positive"\|"negative"\|"mixed"` rather than `string`. | Forces the model to pick from your list and parses straight into the enum. |
| **Keep types small.** Two or three fields are easier for the model than fifteen. | Big types lead to skipped fields and larger output. |
| **Avoid nested types > 2 levels deep.** | Nesting confuses the model and produces noisy output. |
| **Add field descriptions in the type.** Use `# field-name` doc comments. | The descriptions are passed to the model alongside the schema and improve correctness. |

## Source View

Once configured, the generated Ballerina is straightforward enough that you can read and tweak it directly. For an `EmailGenerateResponse` example with bound result `generatedEmail` from provider `emailGenerator`:

```ballerina
import ballerina/ai;

final ai:ModelProvider emailGenerator = check ai:getDefaultModelProvider();

EmailGenerateResponse generatedEmail = check emailGenerator->generate(
    `You are an email writing assistant. Write a short email to ${recipientName}...`
);
```

The `check`-and-bind pattern is what makes the typed binding seamless: BI generates exactly the code you would write by hand, with the schema work already wired up.

## What's Next

- **[The `generate` Node](the-generate-node.md)** — the node these fields live on.
- **[Prompts & Interpolation](prompts-and-interpolation.md)** — back to writing the prompt.
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)** — the same typed-binding idea, packaged as a function.
