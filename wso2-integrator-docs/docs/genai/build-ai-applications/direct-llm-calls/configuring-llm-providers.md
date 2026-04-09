---
sidebar_position: 1
title: "Configuring LLM Providers"
description: "Understand model providers, choose the right LLM, and configure type-safe integration using direct LLM invocation in WSO2 Integrator."
---

# Configuring LLM Providers

Model providers are the gateway between your integration and an LLM. When you make a direct LLM call, you send a prompt along with a type descriptor, and the LLM returns a type-safe response -- a JSON object, a Ballerina record, an integer, or any type you specify. There is no manual parsing. WSO2 Integrator handles serialization and deserialization automatically.

## What is a model provider?

A model provider is a configured connection to an LLM service. It encapsulates the provider's API endpoint, authentication credentials, and model selection. Once configured, you interact with it through a single `generate` API that accepts a prompt and a target type.

```
prompt + type descriptor --> Model Provider --> type-safe response
```

This approach is ideal for simple, stateless interactions where you need structured output without managing conversation history, RAG pipelines, or agent loops.

## Supported providers

WSO2 Integrator supports the following model providers out of the box:

| Provider | Configuration Type | Authentication | Best For |
|---|---|---|---|
| **WSO2 Default** | `ai:Wso2ModelProvider` | Asgardeo (automatic) | Quick prototyping, no API key management |
| **OpenAI** | `ai:OpenAIProvider` | API key | GPT-4o, GPT-4o-mini, reasoning models |
| **Azure OpenAI** | `ai:AzureOpenAIProvider` | API key + endpoint | Enterprise compliance, regional deployments |
| **Anthropic** | `ai:AnthropicProvider` | API key | Claude models, long-context analysis |

:::tip
Start with the **WSO2 Default** provider during development. It requires no API key setup -- authentication is handled through Asgardeo automatically. Switch to a direct provider when you need a specific model, lower latency, or enterprise billing controls.
:::

## Basic direct LLM call

The following example shows how to make a direct LLM call using the default model provider. The `generate` API accepts a prompt and returns the response as the specified type -- either a plain string or a typed record.

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

When you pass a record type like `JokeResponse`, the model provider instructs the LLM to structure its response to match those fields exactly. You get a fully typed record back -- no JSON parsing needed.

## Configuring the model

You configure the model through the connector settings in your `Config.toml`:

```toml
# Using WSO2 default provider (no API key needed)
[ai]
provider = "wso2"

# Using OpenAI directly
[ai]
provider = "openai"
modelId = "gpt-4o-mini"
apiKey = "${OPENAI_API_KEY}"

# Using Azure OpenAI
[ai]
provider = "azure-openai"
apiKey = "${AZURE_OPENAI_API_KEY}"
endpoint = "${AZURE_OPENAI_ENDPOINT}"
deploymentId = "${AZURE_OPENAI_DEPLOYMENT_ID}"
```

:::warning
When using BYOK with external providers, store your API keys securely using WSO2 Integrator's secrets management. Never hard-code keys in source files.
:::

## Choosing a model by task

Different models excel at different tasks. Here is a quick reference:

| Task | Recommended Models | Why |
|---|---|---|
| **Classification** | GPT-4o-mini, Claude Haiku | Low cost, sufficient accuracy for structured output |
| **Summarization** | GPT-4o-mini, Gemini Flash | Cost-effective with strong comprehension |
| **Code generation** | GPT-4o, Claude Sonnet | Strong reasoning and structured output |
| **Conversation** | GPT-4o, Claude Sonnet | Good instruction-following and coherence |
| **Complex reasoning** | o1, GPT-4o | Deep analysis and multi-step logic |

## Cost, latency, and quality trade-offs

| Model | Relative Cost | Avg Latency | Quality | Context Window |
|---|---|---|---|---|
| GPT-4o | High | ~2s | Excellent | 128K |
| GPT-4o-mini | Low | ~0.5s | Good | 128K |
| Claude Sonnet | High | ~2s | Excellent | 200K |
| Claude Haiku | Low | ~0.3s | Good | 200K |
| Gemini Flash | Low | ~0.4s | Good | 1M |

:::tip
Start with a smaller, cheaper model and upgrade only if quality is insufficient. For most classification and extraction tasks, GPT-4o-mini or Claude Haiku deliver excellent results at a fraction of the cost.
:::

## Model fallback strategies

For production integrations, configure fallback models to handle provider outages:

```ballerina
import ballerina/ai;

function generateWithFallback(string prompt) returns string|error {
    ai:ModelProvider|error primaryModel = ai:getDefaultModelProvider();
    if primaryModel is ai:ModelProvider {
        string|error result = primaryModel->generate(prompt);
        if result is string {
            return result;
        }
    }

    // Fall back to secondary provider
    ai:ModelProvider fallbackModel = check ai:getDefaultModelProvider();
    return fallbackModel->generate(prompt);
}
```

## What's next

- [Constructing Prompts](constructing-prompts.md) -- Craft effective prompts for your chosen model
- [Handling Responses](handling-responses.md) -- Handle streaming output, conversation history, and context windows
