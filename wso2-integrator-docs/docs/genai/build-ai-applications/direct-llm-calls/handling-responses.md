---
sidebar_position: 3
title: "Handling Responses"
description: "Handle LLM responses including typed output, conversation history, multimodal input, streaming, and context window management in WSO2 Integrator."
---

# Handling Responses

Once you have configured a model provider and constructed a prompt, you need to handle the LLM's response. This guide covers basic response handling, conversation history, multimodal input, streaming output, and context window management.

## Basic response handling

Direct LLM invocation supports two response modes: plain string responses and typed record responses.

### String responses

The simplest form returns the LLM output as a plain string:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

public function main(string subject) returns error? {
    string joke = check model->generate(`Tell me a joke about ${subject}!`);
    io:println(joke);
}
```

### Typed responses

Pass a Ballerina record type to get structured, validated output:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type JokeResponse record {|
    string setup;
    string punchline;
|};

public function main(string subject) returns error? {
    JokeResponse jokeResponse = check model->generate(`Tell me a joke about ${subject}!`);
    io:println("Setup: ", jokeResponse.setup);
    io:println("Punchline: ", jokeResponse.punchline);
}
```

The model provider instructs the LLM to structure its response to match the specified record fields. If the response does not match the type, an error is returned.

## Conversation history

For multi-turn interactions, use the `chat` API with message history. Each message has a role (`USER` or `ASSISTANT`) and content.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

public function main(string subject) returns error? {
    ai:ChatUserMessage userMessage = {
        role: ai:USER,
        content: `Tell me a joke about ${subject}!`
    };

    ai:ChatMessage[] messages = [userMessage];
    ai:ChatAssistantMessage assistantMessage = check model->chat(messages);
    messages.push(assistantMessage);

    string? joke = assistantMessage?.content;
    io:println(joke);

    messages.push({
        role: ai:USER,
        content: "Explain why this joke is funny."
    });

    ai:ChatAssistantMessage explanation = check model->chat(messages);
    io:println(explanation?.content);
}
```

By appending each message to the `messages` array, you build a conversation history that the LLM uses for context in subsequent calls.

:::note
Direct LLM invocation is stateless -- each `generate` call is independent. Use the `chat` API when you need the LLM to remember previous turns.
:::

## Multimodal input

You can include images alongside text prompts. The LLM processes both the image and the text to produce a typed response.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

type Description record {|
    string description;
    decimal confidence;
    string[] categories;
|};

public function main() returns error? {
    ai:ImageDocument image = {
        content: "https://ballerina.io/img/branding/ballerina_logo_dgrey_png.png"
    };

    Description? description = check model->generate(`
        Describe this image.
        ${image}
        If it is not possible to describe the image, respond with null`);

    if description is null {
        io:println("Could not describe the image.");
    } else {
        io:println("Description: ", description.description);
    }
}
```

The `ai:ImageDocument` type accepts either a URL or base64-encoded image content. The LLM receives both the image and the text prompt, returning a typed response.

## Streaming responses

Streaming lets you send LLM output to clients as it is generated, rather than waiting for the full response. This reduces perceived latency and enables real-time experiences in your integrations.

### When to use streaming

Use streaming when:

- Your integration serves a UI that displays progressive text
- Response generation takes more than a few seconds
- You need to start processing partial output early
- You are building conversational agents with real-time feedback

:::info
For short, structured outputs (classification, extraction), standard non-streaming calls are simpler and sufficient. Streaming adds complexity, so use it only when the latency improvement matters.
:::

### Basic streaming setup

Configure your AI client for streaming and consume tokens as they arrive:

```ballerina
import ballerina/ai;
import ballerina/http;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

service /api on new http:Listener(8080) {

    resource function post chat(@http:Payload ChatRequest req) returns stream<string, error?>|error {
        stream<string, error?> tokenStream = check model->streamGenerate(
            req.message
        );
        return tokenStream;
    }
}

type ChatRequest record {|
    string message;
|};
```

### Server-sent events (SSE) for browsers

For browser-based clients, expose streaming output as server-sent events:

```ballerina
import ballerina/ai;
import ballerina/http;

service /api on new http:Listener(8080) {

    resource function get chat/[string sessionId](http:Caller caller) returns error? {
        http:Response response = new;
        response.setHeader("Content-Type", "text/event-stream");
        response.setHeader("Cache-Control", "no-cache");
        response.setHeader("Connection", "keep-alive");

        stream<string, error?> tokenStream = check model->streamGenerate(
            getPromptForSession(sessionId)
        );

        check caller->respond(response);

        check from string token in tokenStream
            do {
                string sseEvent = string `data: ${token}\n\n`;
                check caller->continue(sseEvent.toBytes());
            };

        check caller->continue("data: [DONE]\n\n".toBytes());
    }
}
```

### Processing streamed output

You can process tokens as they arrive, enabling progressive transformation:

```ballerina
import ballerina/ai;
import ballerina/log;

function streamWithProcessing(string prompt) returns string|error {
    stream<string, error?> tokenStream = check model->streamGenerate(prompt);

    string fullResponse = "";
    int tokenCount = 0;

    check from string token in tokenStream
        do {
            fullResponse += token;
            tokenCount += 1;

            // Log progress every 50 tokens
            if tokenCount % 50 == 0 {
                log:printInfo(string `Received ${tokenCount} tokens...`);
            }
        };

    log:printInfo(string `Stream complete. Total tokens: ${tokenCount}`);
    return fullResponse;
}
```

### Error handling in streams

Streams can fail mid-response. Handle partial output gracefully:

```ballerina
function resilientStream(string prompt) returns string|error {
    stream<string, error?> tokenStream = check model->streamGenerate(prompt);

    string fullResponse = "";

    error? streamError = from string token in tokenStream
        do {
            fullResponse += token;
        };

    if streamError is error {
        if fullResponse.length() > 0 {
            // Return partial response with warning
            return string `[PARTIAL] ${fullResponse}`;
        }
        return streamError;
    }

    return fullResponse;
}
```

:::tip
Log partial responses on stream failure. Even incomplete output can be useful for debugging and may contain enough information for downstream processing.
:::

## Context window management

Every LLM has a context window -- a maximum number of tokens it can process in a single request. Exceeding this limit causes errors or silent truncation that degrades output quality.

### Context window limits by model

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

### Token counting

Estimate token usage before sending requests to avoid hitting limits:

```ballerina
import ballerina/ai;

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

### Truncation strategies

#### Simple truncation

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

#### Sliding window

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

#### Summarization-based truncation

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

    string summary = check model->generate(
        string `Summarize this conversation history concisely, preserving key facts and decisions:\n${olderContent}`
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

## What's next

- [Configuring LLM Providers](configuring-llm-providers.md) -- Choose and configure the right model
- [Constructing Prompts](constructing-prompts.md) -- Design effective prompts for reliable output
