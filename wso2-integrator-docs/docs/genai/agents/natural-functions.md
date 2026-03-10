---
sidebar_position: 7
title: "Use Natural Functions"
description: Define typed function signatures and let the LLM execute the logic at runtime, described in plain language.
---

# Use Natural Functions

Natural functions let you describe logic in plain language and have the LLM execute it at runtime with full type safety. Instead of writing imperative code for tasks like categorization, summarization, or rating, you define the function signature — input types, output types, and a natural language description — and the LLM handles the implementation.

Think of natural functions as typed LLM calls embedded directly in your integration code. They are not agents; they are stateless, single-shot transformations.

## How Natural Functions Work

A natural function has three parts:

1. **Input type** — A Ballerina record that defines the data the function receives
2. **Output type** — A Ballerina record that defines the structure of the response
3. **Natural language description** — Plain English that tells the LLM what to do

At runtime, WSO2 Integrator sends the input data and description to the LLM, then validates and parses the response into the output type.

```
Input (typed) ──► LLM (natural language logic) ──► Output (typed & validated)
```

:::info
Because the LLM generates responses at runtime, natural function outputs may vary slightly across executions. Use specific, constrained output types (enums, bounded integers) to reduce variability.
:::

## Define a Natural Function

Here is a blog review system that categorizes blog posts and generates ratings:

```ballerina
import ballerinax/ai.natural;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;

final openai:Client llmClient = check new ({auth: {token: openaiApiKey}});

// --- Input and Output Types ---

type Blog record {|
    string title;
    string content;
    string author;
|};

type Review record {|
    string[] categories;
    int rating;        // 1-10 scale
    string summary;
    string[] suggestions;
|};

// --- Natural Function ---

@natural:Function {
    llm: llmClient,
    description: string `Analyze the blog post and produce a review:
        - Suggest 1-3 categories from: Technology, Business, Lifestyle, Science, Health, Education
        - Rate the quality from 1 (poor) to 10 (excellent) based on clarity, depth, and originality
        - Write a 2-sentence summary
        - Provide 1-3 improvement suggestions`
}
isolated function reviewBlog(Blog blog) returns Review|error = external;
```

## Use Natural Functions in Services

Call natural functions like any other Ballerina function inside your services:

```ballerina
import ballerina/http;

service /api on new http:Listener(8080) {

    resource function post review(@http:Payload Blog blog) returns Review|error {
        Review review = check reviewBlog(blog);
        return review;
    }
}
```

Test with a request:

```bash
curl -X POST http://localhost:8080/api/review \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Understanding Vector Databases",
    "content": "Vector databases store data as high-dimensional vectors...",
    "author": "Jane Smith"
  }'
```

Example response:

```json
{
  "categories": ["Technology", "Education"],
  "rating": 8,
  "summary": "A comprehensive introduction to vector databases covering core concepts and practical use cases. The article explains embedding generation and similarity search with clear examples.",
  "suggestions": [
    "Add a comparison table of popular vector database options",
    "Include benchmarks for query performance at scale"
  ]
}
```

## Natural Functions vs. Agents vs. Direct LLM Calls

| Aspect | Natural Function | Agent | Direct LLM Call |
|---|---|---|---|
| Stateful | No | Yes | No |
| Typed I/O | Yes | Unstructured | Manual parsing |
| Tool calling | No | Yes | Manual |
| Multi-turn | No | Yes | Manual |
| Best for | Classification, rating, extraction | Conversation, reasoning | Freeform text generation |

Use natural functions when you need **structured, type-safe output** from a single LLM call without the overhead of an agent loop.

## Define Multiple Natural Functions

You can define as many natural functions as your integration needs:

```ballerina
type SentimentResult record {|
    string sentiment;   // "positive", "negative", "neutral"
    float confidence;   // 0.0 to 1.0
|};

@natural:Function {
    llm: llmClient,
    description: "Analyze the sentiment of the given text. Return positive, negative, or neutral with a confidence score."
}
isolated function analyzeSentiment(string text) returns SentimentResult|error = external;

type TranslationResult record {|
    string translatedText;
    string detectedSourceLanguage;
|};

@natural:Function {
    llm: llmClient,
    description: "Translate the given text to the target language."
}
isolated function translateText(string text, string targetLanguage) returns TranslationResult|error = external;

type EntityExtractionResult record {|
    string[] people;
    string[] organizations;
    string[] locations;
    string[] dates;
|};

@natural:Function {
    llm: llmClient,
    description: "Extract named entities from the text: people, organizations, locations, and dates."
}
isolated function extractEntities(string text) returns EntityExtractionResult|error = external;
```

## Compose Natural Functions in Integrations

Chain natural functions together to build transformation pipelines:

```ballerina
import ballerina/http;

type ProcessedArticle record {|
    Blog original;
    Review review;
    SentimentResult sentiment;
    EntityExtractionResult entities;
|};

service /api on new http:Listener(8080) {

    resource function post processArticle(@http:Payload Blog blog) returns ProcessedArticle|error {
        // Run all analyses — these are independent and can execute concurrently
        Review review = check reviewBlog(blog);
        SentimentResult sentiment = check analyzeSentiment(blog.content);
        EntityExtractionResult entities = check extractEntities(blog.content);

        return {
            original: blog,
            review: review,
            sentiment: sentiment,
            entities: entities
        };
    }
}
```

:::tip
Natural functions are stateless and side-effect-free, which makes them safe to run concurrently. Use Ballerina's `start` and `wait` for parallel execution when processing multiple independent transformations.
:::

## Best Practices

- **Constrain output types.** Use enums and bounded ranges (e.g., `int` with documented bounds) to reduce LLM variability.
- **Write specific descriptions.** Vague descriptions lead to inconsistent results. Be explicit about what you expect.
- **Handle errors.** Natural functions can fail if the LLM response does not match the expected type. Always handle the `error` return.
- **Choose the right model.** Simpler tasks (sentiment analysis) can use faster, cheaper models. Complex tasks (detailed reviews) benefit from more capable models.
- **Test for consistency.** Run the same input multiple times during development to understand the output variability range.

:::warning
Natural function outputs are non-deterministic. Do not use them for logic that requires exact, repeatable results (e.g., financial calculations). Use them for tasks where human-like judgment is appropriate.
:::

## What's Next

- [Agent Architecture & Concepts](/docs/genai/agents/architecture-concepts) — Understand when to use natural functions vs. agents
- [Create a Chat Agent](/docs/genai/agents/chat-agents) — Build agents that use natural functions as tools
- [Prompt Engineering for Integrations](/docs/genai/llm-connectivity/prompt-engineering) — Write better descriptions for consistent results
