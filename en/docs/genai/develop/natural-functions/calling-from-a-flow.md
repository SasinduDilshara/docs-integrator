---
sidebar_position: 5
title: Calling from a Flow
description: Use a natural function inside a resource, automation, or another function in WSO2 Integrator BI.
---

# Calling a Natural Function from a Flow

Once a natural function exists in the project, calling it from a flow is a one-step operation. The function appears in the **Add Node** panel under the **AI** category, and calling it works exactly the same as calling any other function.

## Adding the Call

1. Open a resource, automation, or function flow in the **Flow Designer**.
2. Click **+** between two nodes.
3. In the **Add Node** panel, expand the **AI** category and click **Call Natural Function**.

![Add Node panel with the AI section expanded, showing Direct LLM (Model Provider, Call Natural Function), RAG (Knowledge Base, Data Loader, Augment Query). The Call Natural Function tile is highlighted with a tooltip "Call a natural programming function".](/img/genai/develop/natural-functions/32-add-node-call-natural-function.png)

4. The **Natural Functions** picker opens, listing every natural function in the current integration.

![Natural Functions picker showing a Search box and a Current Integration section with the analyzeCustomerReviewes natural function listed.](/img/genai/develop/natural-functions/33-natural-functions-picker.png)

5. Pick the function. The configuration form for that function opens.

## Configuring the Call

The configuration form has one field per parameter, plus a result variable and a (locked) variable type. The field labels match the parameter names you declared on the function — here `CustomerReview` for the `customerReview` parameter:

![Configuration form for the analyzeCustomerReviewes call: CustomerReview field with Text/Expression toggle, Result name reviewResponse, Variable Type ReviewResponse (locked), Save button.](/img/genai/develop/natural-functions/34-call-config-empty.png)

| Field | What it does |
|---|---|
| **\<ParameterName\>** | One row per parameter on the function. Bind each to an in-scope value or expression. The **Text / Expression** toggle on the right switches between literal text and an expression. |
| **Result** | The variable name where the typed return value is stored. |
| **Variable Type** | Locked, pre-filled from the function's declared return type. Override only when you intentionally want a narrower type than the function returns. |

Bind each parameter to the variable you want to send. In the example, `CustomerReview` is bound to the inbound `review` value from the request payload:

![Configuration form filled in: CustomerReview bound to the variable review (shown as a pill), Result reviewResponse, Variable Type ReviewResponse, Save button enabled.](/img/genai/develop/natural-functions/35-call-config-filled.png)

Click **Save**. A new node appears in the flow, named after the bound result variable.

![Resource flow showing Start, the analyzeCustomerReviewes (reviewResponse) node, an empty placeholder, and Error Handler. The Add Node panel on the right is open with Return highlighted under Control.](/img/genai/develop/natural-functions/36-resource-flow-with-call.png)

## Returning the Result

After the natural-function node, add a **Return** node (Control → Return) and bind it to the result variable.

![Return node configuration panel with the message "This operation has no required parameters. Optional settings can be configured below." Expression field set to the variable reviewResponse. A Saving... indicator on the right.](/img/genai/develop/natural-functions/37-return-node-config.png)

The completed flow:

![Final resource flow: Start → analyzeCustomerReviewes (reviewResponse) → Return reviewResponse → Error Handler.](/img/genai/develop/natural-functions/38-final-resource-flow.png)

Because the result is already a typed value, the rest of the flow does not need to know it came from an LLM. You can:

- Return it from an HTTP resource (the typed record becomes the JSON response automatically).
- Pass it into another node — `If`, `Match`, `Map Data`, another function call.
- Persist it via a connector.
- Branch on a field with a `Match` node.

## Calling a Natural Function from Another Function

Natural functions are first-class Ballerina functions. They can call each other.

```ballerina
function pipeline(string ticket) returns Reply|error {
    Triage triage = check triageTicket(ticket);            // a natural function
    Reply reply = check draftReply(triage, ticket);         // another natural function
    return reply;
}
```

Two-step prompts (extract first, then transform) are often cleaner this way than one giant prompt. Each step has its own type, and you can test each one in isolation.

## Calling a Natural Function from an Agent

When you mark a natural function as an [agent tool](/docs/genai/develop/agents/tools), it becomes a button the agent can press during reasoning. The agent decides *whether* to call it; the agent's LLM still does the call. This is a powerful pattern: the agent uses one LLM for routing and another (potentially different) LLM, with a tighter prompt and stricter return type, for a specific subtask.

## Testing the Service

The standard Try-It surface in BI works for natural functions just like any other operation. From an HTTP resource that calls the natural function:

- **Try It** in the top-right of the resource editor sends a request you author by hand.
- **Try it with AI** lets the Copilot check for compile errors, start the service, and send a sample request automatically.

Sample request and response for the `POST /api/v1/analyze` resource shown above (it calls `analyzeCustomerReviewes` and returns the `ReviewResponse`):

```json
// Request
{
  "review": "Loved the sound quality, but the charging case died after a week and support never replied."
}

// Response  (matches the ReviewResponse record type exactly)
{
  "sentiment": "mixed",
  "summary": "Customer loves the sound quality but is frustrated by a faulty charging case and slow support response.",
  "topics": [
    { "name": "sound quality",     "sentiment": "positive" },
    { "name": "charging case",     "sentiment": "negative" },
    { "name": "customer support",  "sentiment": "negative" }
  ],
  "churn_risk": true,
  "suggested_action": "Reach out with a replacement case and apologize for the support delay."
}
```

The fields, types, and structure match the natural function's `ReviewResponse` return type — there is no JSON-parsing layer to write or maintain.

## Common Patterns

### Filter / Branch on the Result

```
Start ─► triage(ticket)
       │
       ├── triage.severity >= 4 ─► escalate
       └── otherwise              ─► acknowledge
```

Use a **Match** or **If** node on the typed return.

### Transform with `Map Data`

A `Map Data` step right after the natural-function call lets you reshape the result for a downstream connector — without a second LLM call.

### Wrap the Function in a Service

A natural function exposed via an HTTP service makes a clean classification or extraction API:

```
POST /reviews/analyze    ─►  analyzeCustomerReviewes(review)  ─►  Return ReviewResponse
POST /tickets/triage     ─►  triageTicket(body)                ─►  Return Triage
POST /emails/extract     ─►  extractEntities(body)             ─►  Return Entities
```

These are some of the highest-leverage uses of natural functions in an integration platform.

## Performance and Cost

Each call is one LLM round-trip — typically a few hundred milliseconds and a few hundred tokens. For high-volume cases:

- **Batch when you can.** A single call on `string[] reviews` is cheaper and faster than calling the function once per review.
- **Cache when the input repeats.** Natural functions are deterministic *enough* at low temperature that caching by input hash is often safe.
- **Measure.** The `bal observe` tooling captures a span for each natural function call alongside the rest of the flow.

## What's Next

- **[Creating a Natural Function](creating-a-natural-function.md)** — build the function this page calls.
- **[The `natural { }` Block](the-natural-block.md)** — back to the function body.
- **[Typed Return Inference](typed-return-inference.md)** — get the most out of the return type.
- **[AI Agents](/docs/genai/develop/agents/overview)** — when natural functions become tools an agent can choose to call.
