---
title: "What Is a Natural Function?"
description: "Understand natural functions -- type-safe LLM execution described in plain language with automatic validation."
---

# What Is a Natural Function?

A natural function is a function where you define the **signature** (inputs and outputs) in code but describe the **logic** in plain language. At runtime, an LLM executes the logic and returns a response that Ballerina validates against your declared types. This gives you the flexibility of natural language with the safety guarantees of a typed programming language.

## The Core Idea

In traditional programming, you write both the function signature and its implementation in code. With natural functions, you split these responsibilities:

- **You** define the input types, output types, and a natural language description of what the function should do.
- **The LLM** interprets your description and generates a response that matches your output type.
- **The runtime** validates the LLM's response against your type definition and returns it as a properly typed value.

```
Input (typed) --> LLM (natural language logic) --> Output (typed & validated)
```

This approach is especially useful for tasks that are hard to express as deterministic code but easy to describe in words -- categorization, summarization, rating, content analysis, and data extraction.

## How Natural Functions Differ from Regular Functions

| Aspect | Regular function | Natural function |
|---|---|---|
| **Logic** | Written in code | Described in natural language |
| **Execution** | Deterministic -- same input always produces same output | Probabilistic -- output may vary slightly across runs |
| **Input/output types** | Defined in code | Defined in code (same type safety) |
| **Best for** | Precise computation, data transformation | Classification, summarization, analysis, extraction |

Because the LLM generates responses at runtime, natural function outputs may vary slightly across executions. Use specific, constrained output types (enums, bounded integers) to reduce variability.

## The `natural` Expression in Ballerina

Ballerina introduces the `natural` expression as a first-class language construct for invoking LLM logic inline. You provide a model provider and a natural language description with template expressions that inject runtime values:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type Attraction record {|
    string name;
    string city;
    string highlight;
|};

function getAttractions(int count, string country, string interest) returns Attraction[]|error {
    Attraction[]|error attractions = natural (model) {
        Give me the top ${count} tourist attractions in ${country}
        for visitors interested in ${interest}.
        For each attraction, the highlight should be one sentence
        describing what makes it special or noteworthy.
    };
    return attractions;
}
```

In this example:

- The return type `Attraction[]` tells the runtime exactly what structure to expect.
- Template expressions like `${count}` and `${country}` inject function parameters into the natural language description.
- The runtime sends the description to the LLM, receives a response, validates it against the `Attraction` record type, and returns a typed array.

## Three Parts of a Natural Function

Every natural function has three components:

1. **Input type** -- A Ballerina record (or parameters) that defines the data the function receives. This data gets injected into the prompt through template expressions.
2. **Output type** -- A Ballerina record that defines the structure of the response. The runtime generates a JSON schema from this type and uses it to constrain the LLM's output.
3. **Natural language description** -- Plain language that tells the LLM what to do. This is the "implementation" of your function.

## Benefits

### Type safety

The output type is enforced at runtime. If the LLM returns a response that does not match your record definition, you get an error rather than silently incorrect data. This makes natural functions safe to use in production integration flows.

### Automatic JSON schema generation

The runtime automatically generates a JSON schema from your Ballerina record type and sends it to the LLM as part of the prompt. You do not need to manually describe the expected output format -- the type system handles it.

### Clean separation of concerns

Business logic described in natural language stays separate from data types and integration plumbing. You can update the prompt without changing the type definitions, and vice versa.

### Reduced boilerplate

Without natural functions, getting structured output from an LLM requires manually constructing prompts, parsing JSON responses, and validating fields. Natural functions reduce this to a type definition and a description.

## When to Use Natural Functions

Natural functions work best for tasks that are:

- **Subjective** -- categorization, rating, sentiment analysis
- **Language-heavy** -- summarization, translation, content generation
- **Hard to codify** -- determining relevance, extracting insights, making recommendations

For tasks that are purely computational or deterministic (math, data transformation, string manipulation), regular functions are more appropriate.

## What's Next

- [What Is an AI Agent?](what-is-an-ai-agent.md) -- Autonomous components that combine LLMs with tools and memory
- [What Is a Large Language Model?](what-is-an-llm.md) -- Understand the LLMs that power natural functions
