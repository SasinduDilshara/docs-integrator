---
sidebar_position: 3
title: "Stream LLM Responses"
description: "Configure streaming from LLMs and handle real-time output in your integration services."
---

# Stream LLM Responses

Streaming lets you send LLM output to clients as it is generated, rather than waiting for the full response. This reduces perceived latency and enables real-time experiences in your integrations. WSO2 Integrator supports streaming through server-sent events (SSE) and chunked responses.

## When to Use Streaming

Use streaming when:

- Your integration serves a UI that displays progressive text
- Response generation takes more than a few seconds
- You need to start processing partial output early
- You are building conversational agents with real-time feedback

:::info
For short, structured outputs (classification, extraction), standard non-streaming calls are simpler and sufficient. Streaming adds complexity, so use it only when the latency improvement matters.
:::

## Basic Streaming Setup

Configure your AI client for streaming and consume tokens as they arrive:

```ballerina
import ballerinax/ai;
import ballerina/http;

final ai:Client aiClient = check new ({model: "gpt-4o"});

service /api on new http:Listener(8080) {

    resource function post chat(@http:Payload ChatRequest req) returns stream<string, error?>|error {
        stream<string, error?> tokenStream = check aiClient->streamGenerate(
            req.message
        );
        return tokenStream;
    }
}

type ChatRequest record {|
    string message;
|};
```

## Server-Sent Events (SSE)

For browser-based clients, expose streaming output as server-sent events:

```ballerina
import ballerinax/ai;
import ballerina/http;

service /api on new http:Listener(8080) {

    resource function get chat/[string sessionId](http:Caller caller) returns error? {
        http:Response response = new;
        response.setHeader("Content-Type", "text/event-stream");
        response.setHeader("Cache-Control", "no-cache");
        response.setHeader("Connection", "keep-alive");

        stream<string, error?> tokenStream = check aiClient->streamGenerate(
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

## Processing Streamed Output

You can process tokens as they arrive, enabling progressive transformation:

```ballerina
import ballerinax/ai;
import ballerina/log;

function streamWithProcessing(string prompt) returns string|error {
    stream<string, error?> tokenStream = check aiClient->streamGenerate(prompt);

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

## Streaming with Timeout

Protect your integrations from hanging streams with timeout handling:

```ballerina
import ballerinax/ai;
import ballerina/lang.runtime;

function streamWithTimeout(string prompt, decimal timeoutSeconds) returns string|error {
    stream<string, error?> tokenStream = check aiClient->streamGenerate(prompt);

    string fullResponse = "";
    boolean timedOut = false;

    future<error?> timer = start watchTimeout(timeoutSeconds);

    check from string token in tokenStream
        do {
            fullResponse += token;
        };

    return fullResponse;
}

function watchTimeout(decimal seconds) returns error? {
    runtime:sleep(seconds);
    return error("Stream timed out");
}
```

:::warning
Always implement timeouts for streaming calls in production. A stalled stream can hold connections open indefinitely and exhaust your service's resources.
:::

## Streaming with Chunked Aggregation

Sometimes you need to buffer streamed tokens into meaningful chunks before forwarding:

```ballerina
function streamInChunks(string prompt, int chunkSize) returns string[]|error {
    stream<string, error?> tokenStream = check aiClient->streamGenerate(prompt);

    string[] chunks = [];
    string currentChunk = "";

    check from string token in tokenStream
        do {
            currentChunk += token;
            if currentChunk.length() >= chunkSize {
                chunks.push(currentChunk);
                currentChunk = "";
            }
        };

    // Push any remaining content
    if currentChunk.length() > 0 {
        chunks.push(currentChunk);
    }

    return chunks;
}
```

## Error Handling in Streams

Streams can fail mid-response. Handle partial output gracefully:

```ballerina
function resilientStream(string prompt) returns string|error {
    stream<string, error?> tokenStream = check aiClient->streamGenerate(prompt);

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

## What's Next

- [Manage Context Windows](context-windows.md) -- Handle token limits in streaming conversations
- [Monitor Performance Metrics](/docs/genai/observability/performance-metrics) -- Track streaming latency and throughput
- [Add Input and Output Guardrails](/docs/genai/guardrails/input-output-guardrails) -- Validate streamed output
