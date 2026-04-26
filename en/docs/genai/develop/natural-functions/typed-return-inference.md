---
sidebar_position: 3
title: Typed Return Inference
description: How the return type of a natural function drives the JSON schema sent to the LLM and the parsing of its response.
---

# Typed Return Inference

The headline feature of natural functions is this: **the return type is the contract**. You declare what you want back; the runtime takes care of telling the model the schema, sending the call, and parsing the response into a typed value.

## How It Works, in Three Steps

When the runtime encounters a `natural { ... }` block, it:

1. Looks at the **return type** of the surrounding function (or the type bound to the result of the expression).
2. Builds a **JSON schema** from that type and includes it in the LLM call.
3. **Parses** the model's response back into a value of that type.

You write the type once. Everything else flows from there.

## The Same Type Choices as `generate`

The full set of return types you can use is the same as for [direct LLM calls](/docs/genai/develop/direct-llm/overview#binding-typed-responses):

| Return type | What you get back |
|---|---|
| `string` | Free-form text. |
| `int`, `decimal`, `boolean` | A scalar parsed from the model's reply. |
| Record types (closed or open `record { ... }`) | A structured value with named, typed fields. |
| Arrays (`Topic[]`) | A list. |
| Unions | The variant the model picked, parsed into the right shape. |

A natural function with a record return type is the most common pattern — it's what makes the function feel like a "smart parser" rather than a "smart string-maker".

## Designing a Good Return Type

The return type is half the prompt. A few rules tend to lift accuracy:

### Use closed records when you want exactly these fields

A **closed record** — declared with the `record &#123;| ... |&#125;` syntax — allows only the listed fields. The schema sent to the model is correspondingly tight, and the parser is strict.

An **open record** — declared with `record { ... }` — tolerates extra fields. Use this only when you genuinely want optionality.

### Use enum unions for fixed-set values

```ballerina
type Sentiment "positive"|"negative"|"neutral";
```

When the type is a union of string singletons, the model is constrained to pick one of those values. You don't need to spell the options out in the prompt — the schema does. The parser ensures you can never receive a value outside the set.

### Add field descriptions

The first line of a doc comment on each field is included in the JSON schema:

```ballerina
type Summary record {|
    # A concise summary of the reviews, under 80 words.
    string summary;
    # Overall sentiment across all reviews.
    "positive"|"negative"|"mixed" sentiment;
    # Key positive themes (top 3-5).
    string[] topPositive;
    # Key negative themes (top 3-5).
    string[] topNegative;
|};
```

The model reads the descriptions and uses them. Field-level constraints in the description (*"under 80 words"*, *"top 3-5"*) tend to be respected even though they aren't enforced by the type.

### Keep nesting shallow

One level of records and arrays is fine. Two levels start to confuse smaller models. Three levels almost always benefit from being split into a flatter shape, or generated in two natural-function steps.

## Why You Don't Need to Say *"Return JSON"*

The `natural { }` body can stay focused on the *task*. The shape comes from the type. Compare:

**Hand-written prompt + `string` + JSON parsing:**

```ballerina
string raw = check model->generate(`Classify the review and return a JSON object
    with fields "sentiment" (one of "positive", "negative", "mixed"), 
    "confidence" (0..1), and "reasons" (string array). Review: ${review}`);
SentimentResult result = check raw.fromJsonStringWithType();
```

**Natural function with the right return type:**

```ballerina
SentimentResult result = natural {
    Classify the review and explain your reasoning.

    Review: ${review}
};
```

Both produce the same `SentimentResult`. The second one is what your code reads like once the type system is doing the heavy lifting.

## Errors and Validation

If the model produces output that does not match the type, the runtime:

1. Asks the model to retry once with a clearer schema reminder.
2. If parsing still fails, returns an `error` from the natural expression.

That means you can rely on the type — by the time the call returns successfully, every required field is present and every value has the right shape. There is no *"the LLM forgot a field"* failure mode further down the flow.

If you want stricter behaviour — for example, *"only accept numeric scores between 1 and 5"* — encode the constraint in the type:

```ballerina
type Score 1|2|3|4|5;
```

Or run a validation step after the call. (Adding stronger constraints in the type usually wins because the model receives them in the schema.)

## A Worked Example

Here is the full picture of how the type drives the system:

```ballerina
type Triage record {|
    # The single best category for this ticket.
    "billing"|"technical"|"account"|"other" category;
    # 1 (informational) to 5 (critical).
    int severity;
    # One-sentence rationale.
    string rationale;
|};

function triageTicket(string ticket) returns Triage|error {
    Triage|error result = natural {
        Triage the support ticket below.

        Ticket:
        ${ticket}
    };
    return result;
}
```

Without you writing a single line of prompt about JSON, the runtime tells the model:

- *"Return a JSON object."*
- *"Field `category` must be one of 'billing', 'technical', 'account', 'other'."*
- *"Field `severity` must be an integer."*
- *"Field `rationale` must be a string."*
- And the descriptions you wrote, verbatim.

The result you receive is a `Triage` record you can branch on, return, persist, or feed into another step.

## What's Next

- **[The `natural { }` block](the-natural-block.md)** — back to the body of the function.
- **[Calling from a Flow](calling-from-a-flow.md)** — invoke the function from a resource or automation.
- **[Binding Typed Responses](/docs/genai/develop/direct-llm/overview#binding-typed-responses)** — same idea applied to direct LLM calls.
