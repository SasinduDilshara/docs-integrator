---
sidebar_position: 4
title: "Manage Context Windows"
description: "Handle token limits, implement truncation strategies, and manage long conversations in LLM-powered integrations."
---

# Manage Context Windows

Every LLM has a context window -- a maximum number of tokens it can process in a single request. Exceeding this limit causes errors or silent truncation that degrades output quality. You need to actively manage context in any integration that handles variable-length input or multi-turn conversations.

## Context Window Limits by Model

| Model | Context Window | Approx. Words | Notes |
|-------|---------------|---------------|-------|
| GPT-4o | 128K tokens | ~96,000 | Good balance of size and cost |
| GPT-4o-mini | 128K tokens | ~96,000 | Cost-effective for large inputs |
| Claude Sonnet | 200K tokens | ~150,000 | Largest among premium models |
| Claude Haiku | 200K tokens | ~150,000 | Fast with large context |
| Gemini Flash | 1M tokens | ~750,000 | Best for very large documents |
| o1 | 128K tokens | ~96,000 | Reserved for reasoning tasks |

:::info
Token counts are approximate. A token is roughly 3/4 of a word in English, but this varies by language and content type. Code and structured data tend to use more tokens per word.
:::

## Token Counting

Estimate token usage before sending requests to avoid hitting limits:

```ballerina
import ballerinax/ai;

function estimateTokens(string text) returns int {
    // Rough estimate: 1 token per 4 characters for English text
    return text.length() / 4;
}

function isWithinLimit(string systemPrompt, string userMessage, int modelLimit) returns boolean {
    int totalTokens = estimateTokens(systemPrompt) + estimateTokens(userMessage);
    // Reserve 25% of context for the response
    int availableTokens = modelLimit * 3 / 4;
    return totalTokens < availableTokens;
}
```

For precise token counting, use the provider's tokenizer:

```ballerina
import ballerinax/ai;

final ai:Client aiClient = check new ({model: "gpt-4o"});

function countTokens(string text) returns int|error {
    ai:TokenCount count = check aiClient->countTokens(text);
    return count.totalTokens;
}
```

## Truncation Strategies

When input exceeds the context window, you need a strategy to reduce it without losing critical information.

### Simple Truncation

Cut from the beginning or end -- suitable for logs and sequential data:

```ballerina
function truncateToFit(string text, int maxTokens) returns string {
    int estimatedTokens = estimateTokens(text);
    if estimatedTokens <= maxTokens {
        return text;
    }

    // Keep the end of the text (most recent content)
    int maxChars = maxTokens * 4;
    return "... [truncated] ..." + text.substring(text.length() - maxChars);
}
```

### Sliding Window

For conversation history, keep the most recent messages plus the system prompt:

```ballerina
type Message record {|
    string role;
    string content;
|};

function applySlidingWindow(Message[] messages, int maxTokens) returns Message[] {
    // Always keep the system prompt (first message)
    Message[] result = [];
    if messages.length() > 0 && messages[0].role == "system" {
        result.push(messages[0]);
    }

    // Add messages from most recent, working backward
    int currentTokens = estimateTokens(result.reduce(
        function(string acc, Message m) returns string => acc + m.content, ""
    ));

    // Iterate from newest to oldest
    foreach int i in 0 ..< messages.length() {
        int idx = messages.length() - 1 - i;
        Message msg = messages[idx];
        if msg.role == "system" {
            continue;
        }
        int msgTokens = estimateTokens(msg.content);
        if currentTokens + msgTokens > maxTokens {
            break;
        }
        result.unshift(msg);
        currentTokens += msgTokens;
    }

    return result;
}
```

### Summarization-Based Truncation

For long conversations, summarize older messages to compress context:

```ballerina
function summarizeOlderMessages(Message[] messages, int splitPoint) returns Message[]|error {
    // Split into old and recent messages
    Message[] olderMessages = messages.slice(1, splitPoint);  // Skip system prompt
    Message[] recentMessages = messages.slice(splitPoint);

    // Summarize older messages
    string olderContent = olderMessages.reduce(
        function(string acc, Message m) returns string =>
            string `${acc}\n${m.role}: ${m.content}`, ""
    );

    string summary = check aiClient->generate(
        string `Summarize this conversation history concisely, preserving key facts and decisions:\n${olderContent}`,
        string
    );

    // Reconstruct with summary
    Message[] result = [messages[0]];  // System prompt
    result.push({role: "system", content: string `Previous conversation summary: ${summary}`});
    result.push(...recentMessages);

    return result;
}
```

:::tip
Summarization-based truncation costs an additional LLM call but preserves much more context than simple truncation. Use it for agent integrations where conversation continuity matters.
:::

## Chunking Large Documents

When processing documents that exceed the context window, split them into overlapping chunks:

```ballerina
function chunkDocument(string document, int chunkSize, int overlap) returns string[] {
    string[] chunks = [];
    int position = 0;

    while position < document.length() {
        int end = int:min(position + chunkSize, document.length());
        chunks.push(document.substring(position, end));
        position += chunkSize - overlap;
    }

    return chunks;
}

function processLargeDocument(string document) returns string[]|error {
    string[] chunks = chunkDocument(document, 8000, 500);
    string[] results = [];

    foreach string chunk in chunks {
        string result = check aiClient->generate(
            string `Extract key findings from this section:\n${chunk}`,
            string
        );
        results.push(result);
    }

    return results;
}
```

:::warning
Overlapping chunks prevent information loss at chunk boundaries, but they increase total token usage and cost. An overlap of 10-15% of the chunk size is a good default.
:::

## Monitoring Context Usage

Track context window utilization to detect issues before they cause failures:

```ballerina
import ballerina/log;

function generateWithContextTracking(string prompt, int modelLimit) returns string|error {
    int promptTokens = estimateTokens(prompt);
    float utilization = <float>promptTokens / <float>modelLimit * 100.0;

    log:printInfo(string `Context utilization: ${utilization.toFixedString(1)}% (${promptTokens}/${modelLimit} tokens)`);

    if utilization > 90.0 {
        log:printWarn("Context window nearly full. Consider truncation or model upgrade.");
    }

    return aiClient->generate(prompt, string);
}
```

## What's Next

- [Model Selection Guide](model-selection.md) -- Compare context windows across models
- [Manage Tokens and Costs](/docs/genai/guardrails/token-cost-management) -- Track and control token spending
- [Prompt Engineering for Integrations](prompt-engineering.md) -- Write concise, effective prompts
