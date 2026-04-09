---
title: "What Is a Large Language Model (LLM)?"
description: "Understand what Large Language Models are, how they work, and how Ballerina connects to them."
---

# What Is a Large Language Model (LLM)?

A Large Language Model (LLM) is a type of artificial intelligence that has been trained on vast amounts of text data to understand and generate human language. LLMs power the AI capabilities throughout WSO2 Integrator, from simple text generation to complex agent reasoning.

## How LLMs Work

LLMs are built on the **transformer architecture**, a neural network design that excels at processing sequential data like text. The lifecycle of an LLM involves two main phases:

### Training

During training, the model processes billions of text documents -- books, articles, websites, code repositories -- and learns statistical patterns about how language works. This includes grammar, facts, reasoning patterns, and even coding conventions. Training happens once (or periodically) and requires enormous computational resources.

The result is a set of **model weights** -- numerical parameters that encode the model's understanding of language. Models like GPT-4, Claude, and Gemini each have billions of these parameters.

### Inference

Inference is what happens when you send a prompt to an LLM and receive a response. The model:

1. **Tokenizes** the input -- breaks your text into smaller units called tokens (roughly 3-4 characters each).
2. **Processes** the tokens through its neural network layers.
3. **Generates** output tokens one at a time, each chosen based on the probability of what should come next given everything before it.

This token-by-token generation is why LLM responses sometimes stream in word by word rather than appearing all at once.

## Key Capabilities

LLMs are general-purpose language processors. The same model can perform many different tasks depending on how you prompt it:

| Capability | Description | Example |
|---|---|---|
| **Text generation** | Produce coherent text on any topic | Draft emails, write documentation, create content |
| **Classification** | Categorize text into predefined groups | Sentiment analysis, ticket routing, content moderation |
| **Summarization** | Condense long text into shorter summaries | Meeting notes, article summaries, report digests |
| **Extraction** | Pull structured data from unstructured text | Extract names, dates, amounts from documents |
| **Translation** | Convert text between languages | Localization, multilingual support |
| **Code generation** | Write and explain programming code | Generate functions, explain algorithms, fix bugs |
| **Reasoning** | Follow logical steps to reach conclusions | Math problems, planning, decision support |

## Structured Output

One of the most powerful capabilities for integration scenarios is **structured output** -- the ability to have an LLM return data in a specific format (like JSON) that your code can parse directly. Rather than processing free-form text, you define the exact shape of the response you expect.

## Connecting to LLMs in Ballerina

WSO2 Integrator provides a unified interface for connecting to LLM providers. You do not need to manage HTTP clients, authentication headers, or response parsing manually. The `ai:ModelProvider` abstraction handles all of that.

Here is a complete example that demonstrates both free-form text generation and structured output from an LLM:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type JokeResponse record {|
    string setup;
    string punchline;
|};

public function main(string subject) returns error? {
    string joke = check model->generate(`Tell me a joke about ${subject}!`);
    io:println(joke);
    JokeResponse jokeResponse = check model->generate(`Tell me a joke about ${subject}!`);
    io:println("Setup: ", jokeResponse.setup);
    io:println("Punchline: ", jokeResponse.punchline);
}
```

In this example:

- `model->generate()` with a `string` return type gives you free-form text.
- `model->generate()` with a `JokeResponse` return type automatically constrains the LLM to return JSON matching the record structure, which Ballerina validates and parses for you.

This type-safe approach to LLM interaction is a core design principle in WSO2 Integrator. It eliminates the need for manual JSON parsing and ensures that LLM outputs conform to your application's data model.

## What's Next

- [What Is a Natural Function?](what-is-a-natural-function.md) -- Type-safe LLM execution through natural language descriptions
- [What Is an AI Agent?](what-is-an-ai-agent.md) -- Autonomous components that reason and act using LLMs
