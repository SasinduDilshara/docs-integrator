---
sidebar_position: 1
title: "Model Selection Guide"
description: "Choose the right LLM for your integration tasks based on cost, latency, quality, and use case requirements."
---

# Model Selection Guide

Selecting the right model for your integration is one of the most impactful decisions you will make. Different models excel at different tasks, and your choice directly affects cost, latency, and output quality. This guide helps you match models to integration use cases in WSO2 Integrator.

## Model Providers

WSO2 Integrator supports multiple model providers through its connector framework. By default, integrations use the **WSO2 Intelligence Endpoint**, which acts as a managed intermediary with zero data retention and Asgardeo-based authentication. You can also connect directly to external providers.

:::info
For connection configuration details (API keys, endpoints, authentication), see the [Connectors section](/docs/connectors/ai-llms).
:::

### Supported Providers

| Provider | Models | Best For |
|----------|--------|----------|
| WSO2 Hosted (default) | GPT-4o, Claude Sonnet | General-purpose, managed access |
| OpenAI | GPT-4o, GPT-4o-mini, o1 | Reasoning, code generation |
| Anthropic | Claude Sonnet, Claude Haiku | Long-context, analysis |
| Azure OpenAI | GPT-4o, GPT-4o-mini | Enterprise compliance |
| Google Vertex AI | Gemini Pro, Gemini Flash | Multimodal, large context |

## Choosing a Model by Task

### Summarization

For summarizing documents, emails, or integration payloads, prioritize models with large context windows and strong comprehension.

```ballerina
import ballerinax/ai;

type Summary record {|
    string mainPoints;
    string actionItems;
    string sentiment;
|};

configurable string modelId = "gpt-4o-mini";

final ai:Client aiClient = check new ({model: modelId});

function summarizeDocument(string document) returns Summary|error {
    return aiClient->generate(
        `Summarize the following document. Extract main points, action items, and overall sentiment: ${document}`,
        Summary
    );
}
```

**Recommended models:** GPT-4o-mini (cost-effective), Claude Haiku (fast), Gemini Flash (large documents).

### Classification

For routing messages, categorizing tickets, or sorting data, you need consistent, structured output.

```ballerina
import ballerinax/ai;

enum TicketCategory {
    BILLING,
    TECHNICAL,
    ACCOUNT,
    GENERAL
}

type ClassificationResult record {|
    TicketCategory category;
    float confidence;
|};

function classifyTicket(string ticketText) returns ClassificationResult|error {
    final ai:Client aiClient = check new ({model: "gpt-4o-mini"});
    return aiClient->generate(
        `Classify this support ticket into exactly one category: ${ticketText}`,
        ClassificationResult
    );
}
```

**Recommended models:** GPT-4o-mini (lowest cost, sufficient accuracy), Claude Haiku (fast classification).

### Code Generation

For generating integration code, transformation logic, or query construction, use models with strong reasoning.

**Recommended models:** GPT-4o (best code quality), Claude Sonnet (strong at structured output), o1 (complex logic).

### Conversation

For agent-based integrations handling multi-turn conversations, prioritize coherence and instruction-following.

**Recommended models:** GPT-4o (balanced), Claude Sonnet (excellent instruction-following), Gemini Pro (large context).

## Cost, Latency, and Quality Trade-offs

| Model | Relative Cost | Avg Latency | Quality | Context Window |
|-------|--------------|-------------|---------|----------------|
| GPT-4o | High | ~2s | Excellent | 128K |
| GPT-4o-mini | Low | ~0.5s | Good | 128K |
| Claude Sonnet | High | ~2s | Excellent | 200K |
| Claude Haiku | Low | ~0.3s | Good | 200K |
| Gemini Flash | Low | ~0.4s | Good | 1M |
| o1 | Very High | ~10s | Excellent (reasoning) | 128K |

:::tip
Start with a smaller, cheaper model and upgrade only if quality is insufficient. For most classification and extraction tasks, GPT-4o-mini or Claude Haiku deliver excellent results at a fraction of the cost.
:::

## Configuring the Model

You configure the model through the connector settings in your `Config.toml`:

```toml
[ai]
modelId = "gpt-4o-mini"
provider = "openai"
apiKey = "${OPENAI_API_KEY}"
```

Or use the WSO2 Intelligence Endpoint for managed access:

```toml
[ai]
provider = "wso2"
# Authentication handled via Asgardeo — no API key needed
```

:::warning
When using the bring-your-own-key option with external providers, ensure your API keys are stored securely using WSO2 Integrator's secrets management. Never hard-code keys in source files.
:::

## Model Fallback Strategies

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

## What's Next

- [Prompt Engineering for Integrations](prompt-engineering.md) -- Craft effective prompts for your chosen model
- [Stream LLM Responses](streaming-responses.md) -- Handle streaming output for real-time integrations
- [Manage Tokens and Costs](/docs/genai/guardrails/token-cost-management) -- Control spending across models
