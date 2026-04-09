---
sidebar_position: 3
title: "Handling Responses"
description: "Constrain output types, handle errors, and choose the right model for natural functions in WSO2 Integrator."
---

# Handling Responses

Natural functions return typed values from LLM calls. The quality and reliability of those values depends on how tightly you constrain the output type, how you handle errors when the LLM response does not match expectations, and which model you choose for the task. This guide covers all three.

## Constrain your output types

Use specific field types and bounded ranges to reduce LLM variability.

- Use **enums** for fields with a fixed set of values instead of open strings.
- Use **bounded integers** (and document the range in the prompt) for rating scales.
- Use **nilable types** (`string?`, `int?`) for fields that may not always be present.
- Use **arrays with documented limits** (e.g., "up to 5 key points") to control output size.

For example, instead of returning `string category`, define the expected values explicitly in the record type and the prompt so the LLM knows exactly what to produce.

## Handle errors gracefully

Natural functions can fail if the LLM response does not match the expected type. Always account for the nilable return or error union.

```ballerina
import ballerina/ai;
import ballerina/log;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type Review record {|
    string suggestedCategory;
    int rating;
|};

function safeReviewBlog(string title, string content) returns Review|error {
    Review?|error result = natural (model) {
        Review the blog post and suggest a category from
        "Technology", "Health", "Travel", "Sports", "Gardening".
        Rate it from 1 to 10.

        Title: ${title}
        Content: ${content}
    };

    if result is error {
        log:printError("Natural function failed", result);
        return result;
    }

    if result is () {
        return error("LLM returned nil -- could not generate a review.");
    }

    return result;
}
```

### Common failure modes

| Failure | Cause | Mitigation |
|---------|-------|------------|
| Type mismatch | LLM returned a string where an int was expected | Use explicit format instructions in the prompt |
| Nil result | LLM could not produce a valid response | Check for `()` and return a meaningful error |
| Timeout | Model took too long to respond | Configure timeouts on the model provider |

## Pick the right model

Not every natural function needs the most capable (and expensive) model. Match the model to the complexity of the task:

- **Simpler tasks** (categorization, extraction) work well with faster, cheaper models like GPT-4o-mini or Claude Haiku.
- **Complex tasks** (multi-criteria evaluation, nuanced summarization) benefit from more capable models like GPT-4o or Claude Sonnet.

You can configure different model providers for different natural functions by passing a different `ai:ModelProvider` instance to each `natural` expression.

## What's next

- [Constructing Prompts](constructing-prompts.md) -- Write effective prompts for classification, extraction, and more
- [Defining Natural Functions](defining-natural-functions.md) -- Learn the syntax and build a complete tutorial
- [Configuring LLM Providers](../direct-llm-calls/configuring-llm-providers.md) -- Choose and configure the right model
