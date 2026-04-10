---
sidebar_position: 1
title: Defining Natural Functions
description: Create LLM-powered typed functions using the natural expression syntax in Ballerina.
---

# Defining Natural Functions

A **natural function** is a regular Ballerina function whose body is -- or contains -- a `natural { ... }` expression. The expression asks an LLM to produce a value that matches the function's declared return type. Because the return type drives the schema sent to the model, you get typed, validated output without manually constructing prompts or parsing JSON.

This is the simplest way to add LLM intelligence to an integration flow: write a function signature, describe the task in natural language, and call the function like any other.

:::warning Experimental feature
Natural expressions are an **experimental** language feature introduced in Ballerina Swan Lake Update 13 Milestone 3 (`2201.13.0-m3`). You must run your program with the `--experimental` flag:

```bash
bal run --experimental
```

The syntax and semantics may change in future releases.
:::

## Anatomy of a Natural Function

A natural function has four parts:

1. A normal Ballerina **function signature** that declares inputs and the return type.
2. An `ai:ModelProvider` value that selects which LLM to call.
3. A `natural (model) { ... }` expression inside the function body.
4. Free-text natural language inside the expression, with `${...}` interpolations for runtime values.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

# Represents a tourist attraction.
type Attraction record {|
    # The name of the attraction
    string name;
    # The city where the attraction is located
    string city;
    # A notable feature or highlight of the attraction
    string highlight;
|};

function getAttractions(int count, string country, string interest)
        returns Attraction[]|error {
    Attraction[]|error attractions =
        natural (model) {
            Give me the top ${count} tourist attractions in ${country}
            for visitors interested in ${interest}.

            For each attraction, the highlight should be one sentence
            describing what makes it special or noteworthy.
        };
    return attractions;
}
```

Calling the function looks exactly like calling any other function:

```ballerina
import ballerina/io;

public function main() returns error? {
    Attraction[] attractions = check getAttractions(3, "Sri Lanka", "Wildlife");
    foreach Attraction attraction in attractions {
        io:println("Name: ", attraction.name);
        io:println("City: ", attraction.city);
        io:println("Highlight: ", attraction.highlight, "\n");
    }
}
```

Run it with:

```bash
bal run --experimental
```

## Choosing the Model

Any `ai:ModelProvider` value can be passed into the `natural (...)` clause. The simplest option is the default WSO2 Copilot provider.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();
```

If you need a specific provider, construct it from the corresponding `ballerinax/ai.*` module. See [Direct LLM Calls](/docs/genai/develop/direct-llm#1-configuring-llm-providers) for the full list.

```ballerina
import ballerina/ai;
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:ModelProvider model = check new openai:ModelProvider(
    openAiApiKey,
    modelType = openai:GPT_4O
);
```

You can also mix providers within a single program -- for example, a cheap model for classification and a powerful one for extraction -- by declaring two `ai:ModelProvider` values and referencing the right one in each natural expression.

```ballerina
final ai:ModelProvider fastModel = check new openai:ModelProvider(
    openAiApiKey,
    modelType = openai:GPT_4O_MINI
);

final ai:ModelProvider reasoningModel = check new openai:ModelProvider(
    openAiApiKey,
    modelType = openai:GPT_4O
);

function classify(string text) returns string|error {
    string category = natural (fastModel) {
        Classify ${text} as one of: billing, technical, shipping, general.
    };
    return category;
}

function extract(string invoice) returns InvoiceSummary|error {
    InvoiceSummary summary = natural (reasoningModel) {
        Extract an invoice summary from: ${invoice}
    };
    return summary;
}
```

## Simple Text-to-Text

When you want a plain string back, declare the return type as `string|error`.

```ballerina
function translate(string text, string targetLanguage) returns string|error {
    string translation = natural (model) {
        Translate the following text to ${targetLanguage}.
        Return only the translation with no extra commentary.

        Text: ${text}
    };
    return translation;
}

// Usage
string french = check translate("Hello, how are you?", "French");
```

## Text to Structured Record

Bind the result to a record type and the runtime infers a JSON schema from the record, including its field documentation.

```ballerina
# Contact information parsed from free text.
type ContactInfo record {|
    # The contact's full name.
    string name;
    # Email address, if present.
    string? email;
    # Phone number, if present.
    string? phone;
    # Company or organization, if present.
    string? company;
|};

function extractContact(string text) returns ContactInfo|error {
    ContactInfo contact = check natural (model) {
        Extract contact information from the text.
        Set missing fields to null.

        Text: ${text}
    };
    return contact;
}

// Usage
ContactInfo contact = check extractContact(
    "Hi, I'm Jane Smith from Acme Corp. Reach me at jane@acme.com or 555-0123."
);
```

## Structured Input and Output

Interpolated values can themselves be records or arrays. The runtime serializes them into a form the model can read.

```ballerina
type Invoice record {|
    string invoiceId;
    string vendor;
    decimal amount;
    string currency;
    string date;
    InvoiceLineItem[] lineItems;
|};

type InvoiceLineItem record {|
    string description;
    decimal amount;
    string category;
|};

# A summary of an invoice suitable for an approver.
type InvoiceSummary record {|
    # The vendor that issued the invoice.
    string vendor;
    # The total amount charged.
    decimal totalAmount;
    # ISO currency code.
    string currency;
    # Number of line items.
    int itemCount;
    # Distinct categories present in the line items.
    string[] categories;
    # True if the total exceeds $10,000 or any line item is consulting.
    boolean requiresApproval;
|};

function analyzeInvoice(Invoice invoice) returns InvoiceSummary|error {
    InvoiceSummary summary = check natural (model) {
        Analyze the invoice below and produce a summary.
        Flag requiresApproval as true if the total exceeds $10,000
        or contains consulting services.

        Invoice: ${invoice}
    };
    return summary;
}
```

## Enum and Union Types

Closed union types give the LLM a fixed set of options to choose from. This is the preferred pattern for classification.

```ballerina
type TicketCategory "billing"|"technical"|"shipping"|"account"|"other";
type TicketPriority "low"|"medium"|"high"|"critical";

type TicketClassification record {|
    TicketCategory category;
    TicketPriority priority;
    string reasoning;
|};

function classifyTicket(string ticketText) returns TicketClassification|error {
    TicketClassification classification = check natural (model) {
        Classify the support ticket into a category and priority level.
        Use 'critical' priority only for service outages or security issues.

        Ticket: ${ticketText}
    };
    return classification;
}
```

## When to Use Natural Functions

| Use Case | Natural Function | Direct `generate` Call | Agent |
|----------|------------------|------------------------|-------|
| Reusable typed transformation | Yes | | |
| One-off inline LLM call | | Yes | |
| Multi-turn conversation | | Yes (`chat`) | Yes |
| Tool calling and memory | | | Yes |

Natural functions shine when the LLM operation is reused across multiple call sites and you want the result to flow through your code as a proper typed value.

## What's Next

- [Constructing Prompts for Natural Functions](/docs/genai/develop/natural-functions/constructing-prompts) -- Write effective text inside `natural { ... }` blocks
- [Handling Natural Function Responses](/docs/genai/develop/natural-functions/handling-responses) -- How return types drive schemas and error handling
- [What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function) -- Conceptual overview
