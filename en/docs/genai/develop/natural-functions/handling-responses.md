---
sidebar_position: 3
title: Handling Natural Function Responses
description: How return types drive JSON schemas for natural expressions and how to handle errors from the LLM.
---

# Handling Natural Function Responses

A natural function is a regular Ballerina function that happens to use a `natural { ... }` expression in its body. That means the *response* is just the function's return value -- typed, validated, and ready to flow through the rest of your integration. This page explains how the return type shapes what the LLM produces, and how to handle the cases where things go wrong.

:::warning Experimental
Natural expressions require Ballerina Swan Lake Update 13 Milestone 3 (`2201.13.0-m3`) or newer, and must be run with `bal run --experimental`.
:::

## The Return Type Drives the Schema

When the Ballerina runtime evaluates a `natural (model) { ... }` expression, it looks at the **expected type** of the expression (usually the function's return type or the variable type on the left-hand side) and converts it into a JSON schema. That schema is sent to the LLM so the model knows exactly what to emit.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

# A tourist attraction.
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

Things to note:

- The return type `Attraction[]|error` tells the runtime "produce an array of `Attraction` values, or propagate an error".
- Field doc comments (`# The name of the attraction`, ...) become descriptions in the JSON schema, so the model understands what each field means.
- The runtime parses the model's JSON output into real Ballerina values before the function returns.

## Error Handling

Any `natural { ... }` expression may fail. Common causes:

- The provider call itself errors (network, auth, rate limit, content policy).
- The model output cannot be coerced into the target type.
- The model refuses or returns an empty response.

Always type the expression as `T|error` (or let the enclosing function return `T|error`) and use `check` or `on fail` to handle failures.

```ballerina
function getAttractionsSafe(int count, string country, string interest)
        returns Attraction[] {
    Attraction[]|error attractions = natural (model) {
        Give me the top ${count} tourist attractions in ${country}
        for visitors interested in ${interest}.
    };

    if attractions is error {
        log:printError("LLM failed to produce attractions",
            'error = attractions,
            country = country
        );
        return [];
    }
    return attractions;
}
```

For more control, capture the result explicitly and branch on the failure.

```ballerina
function classifyWithFallback(string text) returns TicketClassification {
    TicketClassification|error result = natural (model) {
        Classify the support ticket into a category and priority.

        Ticket: ${text}
    };

    if result is error {
        return {
            category: "account",
            priority: "low",
            reasoning: "Classification failed; routed to default queue."
        };
    }
    return result;
}
```

## Optional Results

Include `nil` in the return type when the model may legitimately say "I don't know".

```ballerina
# Description of an image.
type Description record {|
    string description;
    decimal confidence;
    string[] categories;
|};

function describeSubject(string context) returns Description?|error {
    Description? description = check natural (model) {
        Describe the subject based on the context below.
        If it is not possible to describe the subject, respond with null.

        Context: ${context}
    };
    return description;
}
```

## Using Natural Functions in HTTP Services

Because a natural function returns a typed value, you can use it as the response payload of an HTTP resource directly.

```ballerina
import ballerina/ai;
import ballerina/http;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type EmailRequest record {|
    string id;
    string body;
|};

type ClassificationResponse record {|
    string emailId;
    string category;
    string priority;
|};

type TicketClassification record {|
    "billing"|"technical"|"shipping"|"account" category;
    "low"|"medium"|"high"|"critical" priority;
    string reasoning;
|};

function classifyTicket(string ticketText) returns TicketClassification|error {
    TicketClassification classification = check natural (model) {
        Classify the support ticket into a category and priority.

        Ticket: ${ticketText}
    };
    return classification;
}

service /api on new http:Listener(8090) {

    resource function post classify(@http:Payload EmailRequest email)
            returns ClassificationResponse|error {
        TicketClassification classification = check classifyTicket(email.body);
        return {
            emailId: email.id,
            category: classification.category,
            priority: classification.priority
        };
    }
}
```

## Using Natural Functions in Event Handlers

Event-driven services behave the same -- the natural function returns a typed value that you can route on.

```ballerina
import ballerinax/kafka;
import ballerina/log;

type Sentiment "positive"|"negative"|"neutral";

type SentimentResult record {|
    Sentiment sentiment;
    decimal confidence;
|};

function classifySentiment(string feedback) returns SentimentResult|error {
    SentimentResult result = check natural (model) {
        Classify the sentiment of the feedback below.

        Feedback: ${feedback}
    };
    return result;
}

listener kafka:Listener feedbackListener = new ("localhost:9092", {
    groupId: "sentiment-processor",
    topics: ["customer-feedback"]
});

service on feedbackListener {
    remote function onConsumerRecord(kafka:AnydataConsumerRecord[] records)
            returns error? {
        foreach kafka:AnydataConsumerRecord rec in records {
            string feedback = check string:fromBytes(<byte[]>rec.value);
            SentimentResult sentiment = check classifySentiment(feedback);

            if sentiment.sentiment == "negative" && sentiment.confidence > 0.8d {
                log:printWarn("negative feedback detected", feedback = feedback);
            }
        }
    }
}
```

## Using Natural Functions in Data Pipelines

Natural functions compose cleanly with Ballerina's query expressions, so you can run the same LLM operation across a batch of records.

```ballerina
type Document record {|
    string id;
    string content;
|};

type ProcessedDocument record {|
    string id;
    string summary;
    SentimentResult sentiment;
|};

function summarize(string content) returns string|error {
    string summary = check natural (model) {
        Summarize the document in 2-3 sentences.

        Document: ${content}
    };
    return summary;
}

function processBatch(Document[] documents) returns ProcessedDocument[]|error {
    return from Document doc in documents
        select {
            id: doc.id,
            summary: check summarize(doc.content),
            sentiment: check classifySentiment(doc.content)
        };
}
```

## Using Natural Functions in Conditional Logic

Typed return values plug directly into regular Ballerina control flow.

```ballerina
type Department "sales"|"support"|"hr"|"general";

function isSpam(string body) returns boolean|error {
    boolean spam = check natural (model) {
        Is the email below spam or unsolicited marketing?
        Answer true or false.

        Email: ${body}
    };
    return spam;
}

function routeFor(string body) returns Department|error {
    Department dept = check natural (model) {
        Which department should handle the email below?
        Choose one of: sales, support, hr, general.

        Email: ${body}
    };
    return dept;
}

function processIncomingEmail(Email email) returns error? {
    if check isSpam(email.body) {
        check moveToSpam(email);
        return;
    }
    Department dept = check routeFor(email.body);
    check routeEmail(email, dept);
}
```

## Tips for Reliable Responses

- **Prefer closed types.** `"positive"|"negative"|"neutral"` beats `string` for classification because the model cannot invent new categories.
- **Document every field.** Doc comments become part of the schema sent to the LLM, so they act as per-field instructions.
- **Keep the error path explicit.** Natural expressions are just function calls that can fail -- give them the same retry, fallback, and logging treatment you would give any remote call.
- **Lower the temperature.** Construct the `ai:ModelProvider` with a low `temperature` for tasks where consistency matters more than creativity.

## What's Next

- [Defining Natural Functions](/docs/genai/develop/natural-functions/defining) -- Anatomy of a `natural { ... }` expression
- [Constructing Prompts for Natural Functions](/docs/genai/develop/natural-functions/constructing-prompts) -- Write effective text inside `natural { ... }` blocks
- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- When you need multi-step reasoning with tools and memory
