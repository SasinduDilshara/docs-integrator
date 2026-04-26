---
sidebar_position: 6
title: Review Summarizer with Natural Function
description: Step-by-step tutorial — build an HTTP service that uses a natural function to summarize customer reviews and return structured feedback.
---

# Review Summarizer with Natural Function

This tutorial walks through building an **HTTP service that uses a natural function to analyse customer reviews** and return structured feedback. It's an end-to-end scenario for the [Natural Functions](/docs/genai/develop/natural-functions/overview) feature.

By the end you will have a `POST /reviews/summarize` endpoint that takes a list of customer reviews and returns a typed `Summary` containing an overall sentiment, a concise summary, and the top positive and negative themes — all produced by an LLM, using a natural function as the body.

:::caution Experimental Feature
Natural expressions are experimental. BI applies the `--experimental` flag automatically when you click **Run** in the editor.
:::

:::info Default Model Provider for Natural Functions
Before you start, open the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) and run **Configure default model for natural functions (Experimental)**. Sign in with your WSO2 account when prompted. This is a one-time per-project setup.

The first run may also prompt you for the `wso2aiKey` configuration value. The **Configure Application** dialog handles this for you.

![Configure Application dialog prompting for the wso2aiKey model provider configuration.](/img/genai/develop/natural-functions/05-configure-model-provider.png)
:::

## What You'll Build

1. **Define the natural function** with a typed signature and English body.
2. **Create the HTTP resource** that calls the function.
3. **Run and test** the service end to end.

---

## 1. Define the Natural Function

A natural function has two parts: a **typed signature** and a **`natural { ... }` body**.

### Step 1.1 — Create the Function Signature

The function `summarizeCustomerReviews` takes an array of review strings and returns a structured `Summary` record:

| Field | Type | Description |
|---|---|---|
| `summary` | `string` | A concise summary of all reviews. |
| `sentiment` | `string` | Overall sentiment (e.g. `"positive"`, `"negative"`, `"mixed"`). |
| `topPositive` | `string[]` | Key positive themes. |
| `topNegative` | `string[]` | Key negative themes. |

This return type is what drives the LLM's output schema (see [Typed Return Inference](/docs/genai/develop/natural-functions/typed-return-inference)).

### Step 1.2 — Write the Natural Language Body

Instead of writing code inside the function body, you use the `natural` keyword with a plain English description:

```ballerina
function summarizeCustomerReviews(string[] reviews) returns Summary|error {
    Summary|error result = natural {
        Analyze the following customer reviews. Provide a concise summary
        (under 80 words), identify the overall sentiment, and list the
        top positive and negative themes mentioned across all reviews.

        Reviews: ${reviews}
    };
    return result;
}
```

The pieces:

- **`natural { ... }`** — the English body that the LLM evaluates at runtime.
- **`${reviews}`** — interpolates the function's parameter into the prompt. Arrays and records are serialised automatically.
- **Return type `Summary`** — drives the JSON schema. You don't ask the LLM to "return JSON" anywhere.

Once defined, the natural function appears under **Functions** in the project sidebar.

---

## 2. Call the Natural Function from a Resource

### Step 2.1 — Add a Natural Function Call

1. In the resource flow editor, click **+** between **Start** and **Error Handler**.
2. In the **Add Node** panel, under the **AI** section, click **Call Natural Function**.

![Add Node panel showing the AI section with Call Natural Function option.](/img/genai/develop/natural-functions/01-add-node-call-natural-function.png)

3. Pick `summarizeCustomerReviews` from the list and bind its `reviews` parameter to the request payload's `reviews` field.

### Step 2.2 — Complete the Resource Flow

The completed `POST /reviews/summarize` flow is short:

1. **Start** — the request arrives with reviews in the body.
2. **`summarizeCustomerReviews`** — the natural function processes the reviews and returns a `Summary`.
3. **Return** — the `Summary` becomes the HTTP response.

![Resource flow showing Start, summarizeCustomerReviews call, Return, and Error Handler.](/img/genai/develop/natural-functions/02-natural-function-flow.png)

The left sidebar now contains:

- **Functions** > `summarizeCustomerReviews` — the natural function.
- **Types** > `Summary`, plus any sub-types — used by the structured response.
- **HTTP Service** > `POST /reviews/summarize` — the resource that calls the function.

---

## 3. Run and Test

### Step 3.1 — Run the Service

Click **Run** in the editor (or `bal run --experimental` from the terminal). BI applies the `--experimental` flag automatically.

### Step 3.2 — Test with the Copilot

Click **Try it with AI** in the top-right of the resource editor. The Copilot will:

1. Check for compilation errors.
2. Start the service.
3. Send a sample request to `POST /api/v1/reviews/summarize`.

![Copilot testing the POST /reviews/summarize endpoint with a 201 response.](/img/genai/develop/natural-functions/03-test-request.png)

### Step 3.3 — Read the Structured Response

The natural function returns a fully structured response matching `Summary`:

```json
{
  "summary": "Customers appreciated the product's quality and effectiveness, often highlighting its value for money...",
  "sentiment": "positive",
  "topPositive": ["product quality", "effectiveness", "value for money", "customer service"],
  "topNegative": ["shipping delays", "defects", "customer service response time"]
}
```

The Copilot's results panel verifies each field:

- `summary` is a `string`.
- `sentiment` is `"positive"`.
- `topPositive` is an array of strings.
- `topNegative` is an array of strings.

![Test results showing the structured JSON response and all assertions passed.](/img/genai/develop/natural-functions/04-test-results.png)

The LLM correctly analysed the reviews, identified the overall sentiment, extracted positive and negative themes, and produced a concise summary — all returned as a typed `Summary` record. No JSON parsing in your code, no schema enforcement to write by hand.

> Ballerina HTTP services return `201 Created` for POST requests by default — that's expected behaviour, not an error.

---

## What's Next

- **[Typed Return Inference](/docs/genai/develop/natural-functions/typed-return-inference)** — squeeze more accuracy out of the return type.
- **[The `natural { }` Block](/docs/genai/develop/natural-functions/the-natural-block)** — write better English bodies.
- **[Calling from a Flow](/docs/genai/develop/natural-functions/calling-from-a-flow)** — use the function from other places in the project.
- **[Email Generator with Direct LLM](email-generator-direct-llm.md)** — a similar tutorial built around direct LLM calls.
