---
sidebar_position: 4
title: Calling from a Flow
description: Use a natural function inside a resource, automation, or another function in WSO2 Integrator.
---

# Calling a Natural Function from a Flow

Once a natural function exists in the project, calling it from a flow is a one-step operation. The function appears in the **Add Node** panel under the **AI** category, and calling it works exactly the same as calling any other function.

## Adding the Call

1. Open a resource, automation, or function flow in the **Flow Designer**.
2. Click **+** between two nodes.
3. In the **Add Node** panel, expand the **AI** category and click **Call Natural Function**.

![The Add Node panel showing the AI section expanded with the Call Natural Function option.](/img/genai/develop/natural-functions/01-add-node-call-natural-function.png)

4. Pick the natural function you want to call.
5. Map the function's parameters to in-scope variables in the configuration form on the right.
6. Click **Save**.

A new node appears in the flow, named after the bound result variable.

## What the Configuration Form Asks For

| Field | What it does |
|---|---|
| **Function** | The natural function to invoke. The list shows every natural function in the project. |
| **Arguments** | One row per parameter. Bind each parameter to an in-scope value or expression. |
| **Result** | The variable name where the typed return value is stored. |
| **Expected Type** | Pre-filled from the function's declared return type. |

You typically don't need to touch **Expected Type** — the function already declares it. Override it only when you intentionally want a narrower type than the function returns.

## Use the Result Like Any Other Value

After the natural function call, the result variable behaves like any other value:

```
Start ─► reviews (read body)
     │
     ▼
   summarizeReviews(reviews)
     │
     ▼
   Return  (return summary)
```

You can:

- Return it from an HTTP resource (the typed record becomes the JSON response automatically).
- Pass it into another node — `If`, `Match`, `Map Data`, another function call.
- Persist it via a connector.
- Branch on a field with a `Match` node.

Because the result is already a typed value, the rest of the flow does not need to know it came from an LLM.

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

The standard Try-It surface in BI works for natural functions just like any other operation. You can either:

- **Try the surrounding service** — for example, an HTTP resource that calls the natural function. Click **Try It** at the top right.
- **Use the Copilot's Try-it-with-AI** — for an HTTP resource, the Copilot will check for compilation errors, start the service, and send a test request automatically.

After invoking, the Copilot displays the JSON response and validates each field against the declared type:

![Test results panel showing the structured JSON response with assertions on each field of the typed return.](/img/genai/develop/natural-functions/04-test-results.png)

Sample request and response for a review summariser exposed as `POST /api/v1/reviews/summarize`:

```json
// Request
{
  "reviews": [
    "Great product, fast shipping!",
    "Quality is excellent but support was slow."
  ]
}

// Response  (matches the Summary record type exactly)
{
  "summary": "Customers praise quality and shipping speed but flag slow support.",
  "sentiment": "mixed",
  "topPositive": ["product quality", "shipping speed"],
  "topNegative": ["support response time"]
}
```

The fields, types, and structure match the natural function's `Summary` return type — there is no JSON-parsing layer to write or maintain.

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
POST /reviews/summarize  ─►  summarizeReviews(reviews)  ─►  Return Summary
POST /tickets/triage     ─►  triageTicket(body)          ─►  Return Triage
POST /emails/extract     ─►  extractEntities(body)       ─►  Return Entities
```

These are some of the highest-leverage uses of natural functions in an integration platform.

## Performance and Cost

Each call is one LLM round-trip — typically a few hundred milliseconds and a few hundred tokens. For high-volume cases:

- **Batch when you can.** A single call on `string[] reviews` is cheaper and faster than calling the function once per review.
- **Cache when the input repeats.** Natural functions are deterministic *enough* at low temperature that caching by input hash is often safe.
- **Measure.** The `bal observe` tooling captures a span for each natural function call alongside the rest of the flow.

## What's Next

- **[The `natural { }` Block](the-natural-block.md)** — back to the function body.
- **[Typed Return Inference](typed-return-inference.md)** — get the most out of the return type.
- **[AI Agents](/docs/genai/develop/agents/overview)** — when natural functions become tools an agent can choose to call.
