---
sidebar_position: 1
title: "Model Selection Guide"
description: "Understand model providers, choose the right LLM, and build a type-safe integration using direct LLM invocation in WSO2 Integrator."
---

# Model Selection Guide

Model providers are the gateway between your integration and an LLM. When you make a direct LLM call, you send a prompt along with a type descriptor, and the LLM returns a type-safe response -- a JSON object, a Ballerina record, an integer, or any type you specify. There is no manual parsing. WSO2 Integrator handles serialization and deserialization automatically.

This guide explains how model providers work, which providers are available, and walks through a complete tutorial that builds a blog review system using direct LLM invocation.

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

## How direct LLM invocation works

Direct LLM invocation is the simplest way to call an LLM from an integration. You provide:

1. **A prompt** -- natural language instructions for the LLM.
2. **A type descriptor** -- the Ballerina type you want the response to conform to.

The model provider sends the prompt to the LLM, receives the raw response, and maps it to your specified type. If the response does not match the type, an error is returned.

:::note
This is different from agent-based or RAG-based interactions. Direct invocation is stateless -- each call is independent, with no memory of previous calls. Use it for classification, extraction, transformation, and generation tasks.
:::

---

## Tutorial: Build a blog review system

This tutorial walks through building a complete blog review service using the default WSO2 model provider. The service accepts a blog post and returns a structured review with ratings, suggestions, and a summary.

### Step 1: Create a new project

1. Open VS Code and launch the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `BlogReviewer`.

<!-- TODO: Add GIF showing project creation -->

### Step 2: Define the types

Define the input and output types for the review service. The `Blog` type represents the incoming post, and the `Review` type defines the structured review the LLM will generate.

```ballerina
type Blog record {|
    string title;
    string content;
    string author;
|};

type Review record {|
    int rating;             // 1-10 scale
    string summary;         // Brief summary of the blog
    string[] strengths;     // What the blog does well
    string[] improvements;  // Suggested improvements
    string tone;            // e.g., "professional", "casual", "academic"
    boolean publishReady;   // Whether the blog is ready to publish
|};
```

:::note
The `Review` type acts as a contract with the LLM. Because WSO2 Integrator passes this type descriptor to the model provider, the LLM structures its response to match these fields exactly. You get a fully typed `Review` record back -- no JSON parsing needed.
:::

### Step 3: Create the HTTP service

Set up an HTTP service that exposes a review endpoint.

1. Add an HTTP service with base path `/blogs`.
2. Create a **POST** resource named `review`.
3. Set the payload type to `Blog`.
4. Set the return type to `Review`.

<!-- TODO: Add GIF showing HTTP service creation -->

```ballerina
import ballerina/http;
import ballerinax/ai;

service /blogs on new http:Listener(9090) {

    resource function post review(@http:Payload Blog blog) returns Review|error {
        // Implementation in the next step
    }
}
```

### Step 4: Implement the resource logic

Add the model provider and generate the review.

1. Open **Model Provider** in the component palette.
2. Click **+ Add Model Provider**.
3. Select **Default Model Provider (WSO2)**.
4. Name the provider `model` (type: `ai:Wso2ModelProvider`).

<!-- TODO: Add GIF showing model provider setup -->

Now implement the resource function using the `generate` API:

```ballerina
final ai:Wso2ModelProvider model = check new ();

service /blogs on new http:Listener(9090) {

    resource function post review(@http:Payload Blog blog) returns Review|error {
        Review? review = check model->generate(
            string `You are a professional blog editor. Review the following blog post
            and provide a detailed assessment.

            Title: ${blog.title}
            Author: ${blog.author}
            Content: ${blog.content}

            Rate the blog on a scale of 1-10. Identify strengths and areas for improvement.
            Assess the tone and determine if it is ready to publish.`,
            Review
        );

        if review is () {
            return error("Failed to generate review");
        }

        return review;
    }
}
```

<!-- TODO: Add GIF showing generate action setup -->

:::note
The `generate` API returns `Review?` (optional). This accounts for cases where the LLM cannot produce a valid response matching your type. Always handle the `nil` case.
:::

### Step 5: Configure the default WSO2 model provider

1. Open the VS Code Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Run **Ballerina: Configure default WSO2 model provider**.
3. Follow the prompts to authenticate through Asgardeo.

### Step 6: Run and test

Run the integration and send a test request:

```bash
curl -X POST http://localhost:9090/blogs/review \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Getting Started with Microservices",
    "content": "Microservices architecture breaks down applications into small, independent services. Each service handles a specific business function and communicates through APIs. This approach offers better scalability and team autonomy, but introduces complexity in deployment and monitoring.",
    "author": "Jane Smith"
  }'
```

Expected response:

```json
{
  "rating": 6,
  "summary": "A brief introduction to microservices architecture covering core concepts.",
  "strengths": [
    "Clear and concise explanation of the core concept",
    "Acknowledges both benefits and trade-offs"
  ],
  "improvements": [
    "Add concrete examples or code snippets",
    "Expand on deployment and monitoring challenges",
    "Include a comparison with monolithic architecture"
  ],
  "tone": "professional",
  "publishReady": false
}
```

The response is a fully typed `Review` record -- no manual JSON parsing required.

---

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

## Model fallback strategies

For production integrations, configure fallback models to handle provider outages:

```ballerina
import ballerinax/ai;

function generateWithFallback(string prompt) returns string|error {
    ai:Client|error primaryClient = new ({model: "gpt-4o"});
    if primaryClient is ai:Client {
        string|error result = primaryClient->generate(prompt, string);
        if result is string {
            return result;
        }
    }

    // Fall back to secondary provider
    ai:Client fallbackClient = check new ({model: "claude-haiku", provider: "anthropic"});
    return fallbackClient->generate(prompt, string);
}
```

## What's next

- [Prompt Engineering for Integrations](prompt-engineering.md) -- Craft effective prompts for your chosen model
- [Stream LLM Responses](streaming-responses.md) -- Handle streaming output for real-time integrations
- [Context Windows and Token Management](context-windows.md) -- Understand context limits and optimize token usage
- [Manage Tokens and Costs](/docs/genai/guardrails/token-cost-management) -- Control spending across models
