---
sidebar_position: 1
title: "Trace Agent Execution"
description: "Set up end-to-end tracing for agent execution, including LLM calls, tool invocations, and decision paths."
---

# Trace Agent Execution

When an agent processes a request, it makes a series of decisions: choosing which tools to call, reasoning over results, and composing a final response. Tracing captures this entire execution path so you can understand what happened and why. WSO2 Integrator integrates with OpenTelemetry to provide end-to-end visibility into agent behavior.

## What Agent Tracing Captures

A single agent trace includes:

- **User input** -- The original request that triggered the agent
- **LLM calls** -- Each prompt sent, response received, model used, and tokens consumed
- **Tool invocations** -- Which tools the agent called, with what parameters, and what they returned
- **Decision points** -- When the agent chose one action over another
- **Final response** -- The output delivered to the user or downstream system

## Enabling Tracing

Configure OpenTelemetry tracing in your `Config.toml`:

```toml
[ballerina.observe]
tracingEnabled = true
tracingProvider = "jaeger"

[ballerina.observe.tracing.jaeger]
agentHostname = "localhost"
agentPort = 6831
```

:::info
For production tracing setup, including Jaeger deployment and configuration, see the [Tracing guide](/docs/deploy-operate/observe/tracing) in the Deploy and Operate section.
:::

## Adding Custom Spans

Instrument your agent code with custom spans to capture decision points:

```ballerina
import ballerina/observe;
import ballerinax/ai;
import ballerina/log;

final ai:Client aiClient = check new ({model: "gpt-4o"});

function handleAgentRequest(string userInput) returns string|error {
    int startedSpanId = check observe:startSpan("agent.request");

    // Span for LLM reasoning
    int reasoningSpanId = check observe:startSpan("agent.reasoning");
    observe:addTagToSpan(reasoningSpanId, "input.length", userInput.length().toString());

    type AgentDecision record {|
        string selectedTool;
        string reasoning;
        map<string> toolParams;
    |};

    AgentDecision decision = check aiClient->generate(
        string `Given this user request, decide which tool to use and why: ${userInput}`,
        AgentDecision
    );

    observe:addTagToSpan(reasoningSpanId, "selected.tool", decision.selectedTool);
    check observe:finishSpan(reasoningSpanId);

    // Span for tool execution
    int toolSpanId = check observe:startSpan("agent.tool." + decision.selectedTool);
    string toolResult = check executeTool(decision.selectedTool, decision.toolParams);
    observe:addTagToSpan(toolSpanId, "result.length", toolResult.length().toString());
    check observe:finishSpan(toolSpanId);

    // Span for response generation
    int responseSpanId = check observe:startSpan("agent.response");
    string response = check aiClient->generate(
        string `Based on the tool result, compose a response.
                User request: ${userInput}
                Tool: ${decision.selectedTool}
                Result: ${toolResult}`,
        string
    );
    check observe:finishSpan(responseSpanId);

    check observe:finishSpan(startedSpanId);
    return response;
}

function executeTool(string toolName, map<string> params) returns string|error {
    // Tool execution logic
    log:printInfo(string `Executing tool: ${toolName}`);
    return "Tool result placeholder";
}
```

## Tracing Multi-Turn Conversations

For agents that maintain conversation state, trace across multiple turns:

```ballerina
import ballerina/observe;
import ballerina/uuid;

type ConversationTrace record {|
    string conversationId;
    string traceId;
    int turnNumber;
|};

function traceConversationTurn(string conversationId, int turnNumber, string userMessage) returns string|error {
    int spanId = check observe:startSpan("agent.conversation.turn");

    observe:addTagToSpan(spanId, "conversation.id", conversationId);
    observe:addTagToSpan(spanId, "turn.number", turnNumber.toString());
    observe:addTagToSpan(spanId, "user.message.length", userMessage.length().toString());

    string response = check handleAgentRequest(userMessage);

    observe:addTagToSpan(spanId, "response.length", response.length().toString());
    check observe:finishSpan(spanId);

    return response;
}
```

## Visualizing Traces in Jaeger

Once tracing is enabled, open the Jaeger UI to inspect agent execution:

1. Start Jaeger: `docker run -p 16686:16686 -p 6831:6831/udp jaegertracing/all-in-one`
2. Run your integration with tracing enabled
3. Open `http://localhost:16686` in your browser
4. Select your service from the dropdown
5. Click on a trace to see the full execution timeline

You will see a waterfall view showing:

- The total request duration
- Each LLM call with its latency
- Tool execution time
- The proportion of time spent in reasoning vs. tool execution

:::tip
Look for traces where tool execution takes significantly longer than LLM calls. This often indicates slow external services or database queries that you can optimize independently of the AI components.
:::

## Trace-Based Alerts

Set up alerts for abnormal agent behavior:

```ballerina
import ballerina/log;
import ballerina/time;

configurable decimal maxAgentDurationSeconds = 30.0d;
configurable int maxToolCalls = 10;

function monitorAgentExecution(string requestId, decimal durationSeconds, int toolCallCount) {
    if durationSeconds > maxAgentDurationSeconds {
        log:printError(string `ALERT: Agent ${requestId} took ${durationSeconds}s (limit: ${maxAgentDurationSeconds}s)`);
    }

    if toolCallCount > maxToolCalls {
        log:printError(string `ALERT: Agent ${requestId} made ${toolCallCount} tool calls (limit: ${maxToolCalls})`);
    }
}
```

## What's Next

- [Log Agent Conversations](conversation-logging.md) -- Capture full conversation history
- [Monitor Performance Metrics](performance-metrics.md) -- Track latency, tokens, and error rates
- [Debug Agent Behavior](debugging-agent-behavior.md) -- Diagnose common agent failures
