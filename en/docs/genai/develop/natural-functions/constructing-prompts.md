---
sidebar_position: 2
title: Constructing Prompts for Natural Functions
description: Write effective natural language inside natural expression blocks to get reliable, structured LLM output.
---

# Constructing Prompts for Natural Functions

The text you write inside a `natural { ... }` block *is* the prompt that the LLM sees. A well-crafted block is the difference between a reliable integration component and unpredictable output. This page shows the patterns we have found to produce consistent results.

:::warning Experimental
Natural expressions require Ballerina Swan Lake Update 13 Milestone 3 (`2201.13.0-m3`) or newer, and must be run with `bal run --experimental`.
:::

## The Basics

A natural expression has two inputs into the prompt:

1. **Static text** inside the `natural { ... }` body.
2. **Interpolated values** written as `${expression}`.

And one implicit input:

3. **The target type** -- the left-hand side of the assignment or the function's return type. The runtime converts it into a JSON schema that is sent to the model together with your text.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type Sentiment "positive"|"negative"|"neutral";

function classifySentiment(string feedback) returns Sentiment|error {
    Sentiment sentiment = check natural (model) {
        Classify the sentiment of the feedback below.

        Feedback: ${feedback}
    };
    return sentiment;
}
```

Because the target type is the closed union `"positive"|"negative"|"neutral"`, the model is forced into one of those three values.

## Be Specific About the Task

Vague instructions produce vague results. Spell out what the model should do.

```ballerina
// Weak: the model has to guess what "analyze" means
function analyzeInvoiceWeak(Invoice invoice) returns InvoiceSummary|error {
    InvoiceSummary summary = check natural (model) {
        Analyze the invoice: ${invoice}
    };
    return summary;
}

// Strong: the task, the fields, and the rules are explicit
function analyzeInvoice(Invoice invoice) returns InvoiceSummary|error {
    InvoiceSummary summary = check natural (model) {
        Analyze the invoice below and extract:
        - vendor name
        - total amount
        - currency
        - line item count
        - whether it requires approval

        Flag requiresApproval as true if the total exceeds $10,000
        or any line item is categorized as consulting.

        Invoice: ${invoice}
    };
    return summary;
}
```

## Let the Return Type Carry the Shape

You do not need to describe the output JSON in the prompt -- the runtime already ships a schema derived from the return type. Instead, describe the **meaning** of each field in a doc comment on the record.

```ballerina
# A customer-facing summary of an order issue.
type OrderIssueSummary record {|
    # One sentence that states the problem in plain language.
    # Do not mention internal system names or error codes.
    string issue;
    # Empathetic acknowledgement line for the customer.
    string acknowledgement;
    # The next step the customer should take.
    string nextStep;
|};

function summarizeOrderIssue(OrderIssue issue) returns OrderIssueSummary|error {
    OrderIssueSummary summary = check natural (model) {
        Build a short, empathetic summary for the customer.
        Keep each field under 25 words.

        Issue: ${issue}
    };
    return summary;
}
```

Doc comments on record fields are included in the schema, so the LLM sees them as part of the instructions.

## Define Classification Criteria

For classification, combine a closed union type with explicit definitions of each value.

```ballerina
type TicketCategory "billing"|"technical"|"shipping"|"account";

type TicketClassification record {|
    TicketCategory category;
    string reasoning;
|};

function classifyTicket(string ticketText) returns TicketClassification|error {
    TicketClassification classification = check natural (model) {
        Classify the support ticket into exactly one category.

        Categories:
        - billing: payment, invoice, or refund issues
        - technical: API errors, integration failures, configuration problems
        - shipping: delivery, tracking, damaged items
        - account: login, profile, or permissions

        If unclear, choose the most likely category based on key terms.

        Ticket: ${ticketText}
    };
    return classification;
}
```

## Include Output Constraints

Use the natural block to enforce editorial rules that the type system cannot express.

```ballerina
function generateCustomerSummary(OrderIssue issue) returns string|error {
    string summary = check natural (model) {
        Generate a customer-facing summary of the order issue.

        Requirements:
        1. Do not mention internal system names or error codes.
        2. Do not promise specific resolution timelines.
        3. Keep the summary under 100 words.
        4. Use empathetic, professional language.

        Issue: ${issue}
    };
    return summary;
}
```

## Chain-of-Thought for Complex Tasks

For multi-step reasoning, instruct the model to think through the problem before producing the final value.

```ballerina
type ComplaintResolution record {|
    "refund"|"replacement"|"credit"|"escalation" recommendation;
    string reasoning;
|};

function analyzeComplaint(string complaint) returns ComplaintResolution|error {
    ComplaintResolution resolution = check natural (model) {
        Analyze the customer complaint and determine the appropriate resolution.

        Think step by step:
        1. Identify the core issue.
        2. Determine which product or service is affected.
        3. Check if this matches a known issue pattern.
        4. Recommend a resolution category: refund, replacement, credit, or escalation.

        Complaint: ${complaint}
    };
    return resolution;
}
```

The `reasoning` field in the return type gives the model somewhere to "show its work" without bloating the final recommendation.

## Interpolating Multiple Values

You can interpolate any number of expressions. Pass complex objects directly -- the runtime handles serialization.

```ballerina
type FollowUpEmail record {|
    string subject;
    string body;
|};

function draftFollowUp(CustomerProfile profile, Order recentOrder)
        returns FollowUpEmail|error {
    FollowUpEmail email = check natural (model) {
        Draft a personalized follow-up email for the customer below.
        Tone: friendly and professional.
        Mention at least one specific item from the recent order.
        Keep the body under 150 words.

        Customer profile: ${profile}
        Recent order: ${recentOrder}
    };
    return email;
}
```

## Reusable Prompt Building Blocks

Because the body of a natural expression is free text, you can factor repeated framing into a helper function that returns a plain string and interpolate it.

```ballerina
function toneGuidance(string channel) returns string {
    match channel {
        "email" => { return "Formal, third-person, no emojis."; }
        "chat" => { return "Casual, second-person, concise."; }
        _ => { return "Neutral, professional."; }
    }
}

function draftReply(string message, string channel) returns string|error {
    string tone = toneGuidance(channel);
    string reply = check natural (model) {
        Draft a reply to the customer message below.
        Tone rules: ${tone}

        Message: ${message}
    };
    return reply;
}
```

## Testing Prompts

Because a natural function is just a Ballerina function, you can test it with the standard `ballerina/test` module.

```ballerina
import ballerina/test;

@test:Config {}
function testTicketClassification() returns error? {
    TicketClassification result = check classifyTicket(
        "My API key isn't working and I get a 401 error"
    );
    test:assertEquals(result.category, "technical");

    result = check classifyTicket("I was charged twice for my subscription");
    test:assertEquals(result.category, "billing");

    result = check classifyTicket("My package hasn't arrived and it's been two weeks");
    test:assertEquals(result.category, "shipping");
}
```

When a test fails, iterate on the prompt by:

1. Tightening the return type into a closed union where possible.
2. Adding concrete definitions for each category or field.
3. Lowering the provider's `temperature` for more deterministic output.
4. Including a short example in the natural block for tricky edge cases.

## What's Next

- [Defining Natural Functions](/docs/genai/develop/natural-functions/defining) -- Anatomy of a `natural { ... }` expression
- [Handling Natural Function Responses](/docs/genai/develop/natural-functions/handling-responses) -- How return types drive schemas and error handling
- [Constructing Prompts (Direct LLM)](/docs/genai/develop/direct-llm#2-constructing-prompts) -- Prompt engineering for `generate` and `chat`
