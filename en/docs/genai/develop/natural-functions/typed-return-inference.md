---
sidebar_position: 4
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

## Setting the Return Type in BI

The return type is set on the **Create New Natural Function** form, in the **Return Type** field. The picker offers primitives, plus **Create New Type** and **Open Type Browser**.

![Return Type dropdown listing Primitive Types (string, int, float, decimal, boolean), with a Create New Type entry and an Open Type Browser link at the bottom.](/img/genai/develop/natural-functions/14-return-type-dropdown.png)

For most natural functions you will want a **record** so each output field is named and typed independently.

### Create New Type — From Scratch

The **Create from scratch** tab gives you full control over field names, types, and descriptions.

![Create New Type dialog on the Create from scratch tab, with Kind set to Record, Name field reading ReviewResponse, an empty Fields section with a + button, and Advanced Options collapsed.](/img/genai/develop/natural-functions/15-create-type-scratch.png)

| Field | What it does |
|---|---|
| **Kind** | `Record`, `Enum`, `Union`, `Array`. `Record` is the common choice. |
| **Name** | The type identifier as it will appear in the project (e.g. `ReviewResponse`). |
| **Fields** | Click **+** to add each field with name, type, and an optional description. The description ends up in the JSON schema sent to the LLM. |
| **Advanced Options** | Closed/open record toggle, defaults, and other type-level switches. |

### Create New Type — Import JSON

If you already have a sample of the JSON shape you want, switch to the **Import** tab. BI infers the record type, including nested records and arrays.

![Create New Type dialog on the Import tab, with Format set to JSON, Name set to ReviewResponse, an Import JSON File button, and a textarea showing a pasted JSON sample with sentiment, summary, topics, churn_risk, and suggested_action fields. The Import button is at the bottom right.](/img/genai/develop/natural-functions/16-create-type-import-json.png)

| Field | What it does |
|---|---|
| **Format** | `JSON` (other structured formats supported too). |
| **Name** | Top-level type name to generate. |
| **Import JSON File** | Loads a `.json` file from disk. Or paste the sample directly into the textarea. |

For example, the sample above produces:

- `ReviewResponse` with `sentiment`, `summary`, `topics`, `churn_risk`, `suggested_action`.
- `Topics` (the array element record) with `name` and `sentiment`.

The new type and any sub-types are added under **Types** in the project sidebar and selected as the function's return type:

![Create New Natural Function form with Name analyzeCustomerReviewes, parameter pill "string customerReview", Return Type ReviewResponse, and Create button enabled. Sidebar Types tree shows ReviewResponse, Topics, TopicsItem.](/img/genai/develop/natural-functions/17-create-form-filled.png)

You don't have to worry about declaring schemas separately — the same record drives both the value you receive and the schema the LLM is constrained to.

## The Same Type Choices as `generate`

The full set of return types you can use is the same as for [direct LLM calls](/docs/genai/develop/direct-llm/overview#binding-typed-responses):

| Return type | What you get back |
|---|---|
| `string` | Free-form text. |
| `int`, `decimal`, `boolean` | A scalar parsed from the model's reply. |
| Record types (closed or open `record { ... }`) | A structured value with named, typed fields. |
| Arrays (`Topic[]`) | A list. |
| Unions | The variant the model picked, parsed into the right shape. |

A natural function with a record return type is the most common pattern — it's what makes the function feel like a *smart parser* rather than a *smart string-maker*.

## Designing a Good Return Type

The return type is half the prompt. A few rules tend to lift accuracy.

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
type ReviewResponse record {|
    # The overall sentiment of the review.
    "positive"|"negative"|"mixed"|"neutral" sentiment;
    # A concise summary of the review, under 80 words.
    string summary;
    # Per-topic sentiment for each key topic raised in the review.
    Topic[] topics;
    # True if the customer is at risk of churning.
    boolean churn_risk;
    # A specific follow-up action the support team should take.
    string suggested_action;
|};
```

The model reads the descriptions and uses them. Field-level constraints in the description (*"under 80 words"*, *"top 3-5"*) tend to be respected even though they aren't enforced by the type.

The same descriptions are also surfaced in the BI Type editor — they show in the Fields list, the **Add Parameter** form when callers bind values, and the JSON schema the LLM sees. One source of truth.

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
type Topic record {|
    # The name of the topic raised in the review.
    string name;
    # The sentiment expressed about that topic.
    "positive"|"negative"|"neutral" sentiment;
|};

type ReviewResponse record {|
    "positive"|"negative"|"mixed"|"neutral" sentiment;
    string summary;
    Topic[] topics;
    boolean churn_risk;
    string suggested_action;
|};

function analyzeCustomerReviewes(string customerReview) returns ReviewResponse|error {
    ReviewResponse|error result = natural {
        You are a **customer review analyzer**. Identify the overall sentiment,
        extract the key topics being discussed with their individual sentiment,
        and suggest a follow-up action if needed.

        Review: ${customerReview}
    };
    return result;
}
```

Without you writing a single line of prompt about JSON, the runtime tells the model:

- *"Return a JSON object."*
- *"Field `sentiment` must be one of 'positive', 'negative', 'mixed', 'neutral'."*
- *"Field `topics` is an array of `{name, sentiment}` records, with `sentiment` from the same enum."*
- *"Field `churn_risk` must be a boolean."*
- *"Field `suggested_action` must be a string."*
- And the descriptions you wrote, verbatim.

The result you receive is a `ReviewResponse` record you can branch on, return, persist, or feed into another step.

## What's Next

- **[The `natural { }` block](the-natural-block.md)** — back to the body of the function.
- **[Calling from a Flow](calling-from-a-flow.md)** — invoke the function from a resource or automation.
- **[Binding Typed Responses](/docs/genai/develop/direct-llm/overview#binding-typed-responses)** — same idea applied to direct LLM calls.
