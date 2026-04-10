---
sidebar_position: 2
title: What is a Natural Function?
description: Understand natural functions -- LLM-powered typed function calls in Ballerina.
---

# What is a Natural Function?

A natural function is a Ballerina function whose body is a **natural expression** -- a block of natural language evaluated by an LLM at runtime. You write the function signature in the usual way and describe the behavior in plain English inside a `natural { ... }` block. The compiler handles prompt construction, LLM invocation, and response parsing against the declared return type.

Natural expressions are the simplest way to add LLM intelligence to your integrations without building a full agent.

> **Note:** Natural expressions are an experimental language feature. Run your program with `bal run --experimental` (Ballerina Swan Lake Update 13 Milestone 3 or later).

## How Natural Functions Work

1. You declare a function with a typed signature and describe the task inside a `natural (model) { ... }` expression
2. The Ballerina compiler derives the prompt and the expected output schema from the return type
3. At runtime, the expression sends the prompt to the configured LLM
4. The LLM's response is parsed and returned as a typed Ballerina value

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type SentimentResult record {|
    "positive"|"negative"|"neutral" sentiment;
    float confidence;
    string explanation;
|};

function classifySentiment(string text) returns SentimentResult|error {
    SentimentResult result = natural (model) {
        Classify the sentiment of the following text as positive, negative,
        or neutral. Provide a confidence score between 0 and 1 and a short
        explanation.

        Text: ${text}
    };
    return result;
}
```

You call it like any other function:

```ballerina
SentimentResult result = check classifySentiment("This product exceeded my expectations!");
// result.sentiment == "positive"
```

## When to Use Natural Functions

| Scenario | Use Natural Functions | Use Agents |
|----------|:--------------------:|:----------:|
| Single-step text transformation | Yes | |
| Classification or categorization | Yes | |
| Data extraction from text | Yes | |
| Multi-step reasoning with tools | | Yes |
| Conversational interactions | | Yes |
| Tasks requiring memory | | Yes |

Natural functions are best for **single-step transformations** where you need the LLM to process an input and return a typed result without calling tools or maintaining state.

## Key Characteristics

- **Typed** -- Input and output types are enforced by Ballerina's type system, and the return type drives the LLM's output schema
- **Stateless** -- No memory or conversation history
- **No tool calling** -- The LLM cannot call external functions
- **Reusable** -- Defined once, callable from anywhere in your code
- **Testable** -- Can be mocked and tested like any other function

## What's Next

- [What is an AI Agent?](what-is-ai-agent.md) -- For multi-step reasoning with tools
- [Defining Natural Functions](/docs/genai/develop/natural-functions/defining) -- Detailed implementation guide
- [Constructing Prompts for Natural Functions](/docs/genai/develop/natural-functions/constructing-prompts) -- Write effective descriptions
