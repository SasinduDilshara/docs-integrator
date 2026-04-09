---
sidebar_position: 2
title: "Constructing Prompts"
description: "Learn how to write effective prompts for natural functions, with examples for classification, data extraction, summarization, and sentiment analysis."
---

# Constructing Prompts

Natural functions are well suited for tasks where the logic is easier to describe in natural language than to implement procedurally. The quality of your results depends heavily on how you write the prompt inside the `natural` expression. This guide covers common patterns, prompt structure best practices, and worked examples.

## Prompt structure guidelines

Follow these guidelines to get consistent, high-quality results from natural functions:

- **List the exact categories, scales, or formats you expect.** Vague instructions lead to inconsistent results.
- **Specify what to do when data is missing** (e.g., "use null").
- **Define boundaries clearly** (e.g., "on a scale of 1 to 10", "in 2-3 sentences").
- **Use numbered tasks** if the function must perform multiple operations.
- **Match the prompt to the output type.** The fields in your record type should correspond directly to what you ask the LLM to produce.

## Common use cases

### Classification

Natural functions excel at classifying input into predefined categories. Define the categories as an enum or constrained string type, and describe the classification criteria in the prompt.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type ClassificationResult record {|
    string category;
    decimal confidence;
|};

function classifyTicket(string ticketDescription) returns ClassificationResult|error {
    ClassificationResult|error result = natural (model) {
        Classify the following support ticket into one of these categories:
        "billing", "technical", "account", "general".
        Also provide a confidence score between 0.0 and 1.0.

        Ticket: ${ticketDescription}
    };
    return result;
}
```

### Data extraction

Extract structured data from unstructured text. The output type defines exactly which fields to extract.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type ContactInfo record {|
    string? name;
    string? email;
    string? phone;
    string? company;
|};

function extractContact(string emailBody) returns ContactInfo|error {
    ContactInfo|error contact = natural (model) {
        Extract contact information from the following email.
        If a field is not found, use null.

        Email: ${emailBody}
    };
    return contact;
}
```

### Summarization

Summarize long-form content into a structured format with specific constraints.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type Summary record {|
    string title;
    string summary;
    string[] keyPoints;
|};

function summarizeArticle(string article) returns Summary|error {
    Summary|error summary = natural (model) {
        Summarize the following article.
        The summary should be 2-3 sentences.
        Extract up to 5 key points as short bullet items.

        Article: ${article}
    };
    return summary;
}
```

### Sentiment analysis

Analyze the sentiment of text input and return a structured assessment.

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type SentimentResult record {|
    string sentiment;
    decimal score;
    string explanation;
|};

function analyzeSentiment(string text) returns SentimentResult|error {
    SentimentResult|error result = natural (model) {
        Analyze the sentiment of the following text.
        Classify the sentiment as "positive", "negative", or "neutral".
        Provide a score between -1.0 (most negative) and 1.0 (most positive).
        Give a brief one-sentence explanation.

        Text: ${text}
    };
    return result;
}
```

## What's next

- [Handling Responses](handling-responses.md) -- Constrain output types and handle errors in natural functions
- [Defining Natural Functions](defining-natural-functions.md) -- Learn the syntax and build a complete tutorial
- [Configuring LLM Providers](../direct-llm-calls/configuring-llm-providers.md) -- Choose and configure the right model for your natural functions
