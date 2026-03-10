---
sidebar_position: 3
title: "Monitor Performance Metrics"
description: "Track LLM call latency, tool execution time, token consumption, and agent success rates for AI-powered integrations."
---

# Monitor Performance Metrics

Performance monitoring tells you how your AI-powered integrations are behaving in production. You need to track LLM call latency, tool execution time, token consumption, error rates, and agent success rates. WSO2 Integrator exposes these metrics through its observability framework, and you can push them to your preferred monitoring stack.

## Key Metrics to Track

| Metric | What It Measures | Why It Matters |
|--------|-----------------|----------------|
| LLM call latency | Time from request to response for each LLM call | Directly impacts user experience |
| Tool execution time | Duration of each tool invocation by an agent | Identifies slow external dependencies |
| Tokens consumed | Input + output tokens per request | Controls cost |
| Error rate | Percentage of failed LLM calls or agent runs | Signals reliability issues |
| Agent success rate | Percentage of agent runs that completed their task | Measures overall effectiveness |
| Time to first token | Latency before streaming begins | Key for streaming UX |

## Collecting Metrics

Instrument your integration to capture metrics on every LLM interaction:

```ballerina
import ballerina/time;
import ballerina/log;
import ballerinax/ai;

type LlmMetrics record {|
    string requestId;
    string model;
    decimal latencyMs;
    int promptTokens;
    int completionTokens;
    boolean success;
    string? errorType;
|};

final ai:Client aiClient = check new ({model: "gpt-4o-mini"});

function generateWithMetrics(string requestId, string prompt) returns [string, LlmMetrics]|error {
    time:Utc startTime = time:utcNow();

    string|error result = aiClient->generate(prompt, string);

    time:Utc endTime = time:utcNow();
    decimal latencyMs = <decimal>(endTime[0] - startTime[0]) * 1000d;

    LlmMetrics metrics = {
        requestId: requestId,
        model: "gpt-4o-mini",
        latencyMs: latencyMs,
        promptTokens: prompt.length() / 4,  // Estimate
        completionTokens: 0,
        success: result is string,
        errorType: result is error ? result.message() : ()
    };

    // Publish metrics
    publishMetrics(metrics);

    if result is error {
        return result;
    }

    return [result, metrics];
}

function publishMetrics(LlmMetrics metrics) {
    log:printInfo(string `LLM_METRIC | request=${metrics.requestId} | model=${metrics.model} | ` +
        string `latency=${metrics.latencyMs}ms | tokens=${metrics.promptTokens + metrics.completionTokens} | ` +
        string `success=${metrics.success}`);
}
```

## Agent-Level Metrics

Track metrics across the full agent execution, not just individual LLM calls:

```ballerina
type AgentRunMetrics record {|
    string agentId;
    string runId;
    decimal totalDurationMs;
    int llmCallCount;
    int toolCallCount;
    decimal totalLlmLatencyMs;
    decimal totalToolLatencyMs;
    int totalTokens;
    boolean success;
    string? failureReason;
|};

class MetricsCollector {
    private string agentId;
    private string runId;
    private time:Utc startTime;
    private int llmCalls = 0;
    private int toolCalls = 0;
    private decimal llmLatency = 0d;
    private decimal toolLatency = 0d;
    private int tokens = 0;

    function init(string agentId, string runId) {
        self.agentId = agentId;
        self.runId = runId;
        self.startTime = time:utcNow();
    }

    function recordLlmCall(decimal latencyMs, int tokensUsed) {
        self.llmCalls += 1;
        self.llmLatency += latencyMs;
        self.tokens += tokensUsed;
    }

    function recordToolCall(decimal latencyMs) {
        self.toolCalls += 1;
        self.toolLatency += latencyMs;
    }

    function finalize(boolean success, string? failureReason = ()) returns AgentRunMetrics {
        time:Utc endTime = time:utcNow();
        decimal totalDuration = <decimal>(endTime[0] - self.startTime[0]) * 1000d;

        AgentRunMetrics metrics = {
            agentId: self.agentId,
            runId: self.runId,
            totalDurationMs: totalDuration,
            llmCallCount: self.llmCalls,
            toolCallCount: self.toolCalls,
            totalLlmLatencyMs: self.llmLatency,
            totalToolLatencyMs: self.toolLatency,
            totalTokens: self.tokens,
            success: success,
            failureReason: failureReason
        };

        log:printInfo(string `AGENT_METRIC | agent=${metrics.agentId} | run=${metrics.runId} | ` +
            string `duration=${metrics.totalDurationMs}ms | llm_calls=${metrics.llmCallCount} | ` +
            string `tool_calls=${metrics.toolCallCount} | tokens=${metrics.totalTokens} | ` +
            string `success=${metrics.success}`);

        return metrics;
    }
}
```

## Exposing a Metrics Endpoint

Expose metrics for scraping by Prometheus or other monitoring tools:

```ballerina
import ballerina/http;

// In-memory metric aggregations
isolated int totalRequests = 0;
isolated int failedRequests = 0;
isolated decimal totalLatency = 0d;
isolated int totalTokensConsumed = 0;

function updateAggregates(LlmMetrics metrics) {
    lock {
        totalRequests += 1;
        if !metrics.success {
            failedRequests += 1;
        }
        totalLatency += metrics.latencyMs;
        totalTokensConsumed += metrics.promptTokens + metrics.completionTokens;
    }
}

service /metrics on new http:Listener(9090) {

    resource function get .() returns string {
        lock {
            decimal avgLatency = totalRequests > 0 ? totalLatency / <decimal>totalRequests : 0d;
            float errorRate = totalRequests > 0 ? <float>failedRequests / <float>totalRequests : 0.0;

            return string `# HELP llm_requests_total Total LLM requests
# TYPE llm_requests_total counter
llm_requests_total ${totalRequests}

# HELP llm_requests_failed Total failed LLM requests
# TYPE llm_requests_failed counter
llm_requests_failed ${failedRequests}

# HELP llm_latency_avg_ms Average LLM call latency in milliseconds
# TYPE llm_latency_avg_ms gauge
llm_latency_avg_ms ${avgLatency}

# HELP llm_tokens_total Total tokens consumed
# TYPE llm_tokens_total counter
llm_tokens_total ${totalTokensConsumed}

# HELP llm_error_rate LLM call error rate
# TYPE llm_error_rate gauge
llm_error_rate ${errorRate}
`;
        }
    }
}
```

:::info
For full observability stack setup (Prometheus, Grafana, alerting), see the [Metrics guide](/docs/deploy-operate/observe/metrics) in the Deploy and Operate section.
:::

## Setting Up Alerts

Define alert thresholds for critical metrics:

```ballerina
configurable decimal latencyAlertThresholdMs = 5000d;
configurable float errorRateAlertThreshold = 0.05;
configurable int tokenBudgetPerHour = 1000000;

function checkAlerts(AgentRunMetrics metrics) {
    if metrics.totalDurationMs > latencyAlertThresholdMs {
        log:printError(string `LATENCY_ALERT: Agent ${metrics.agentId} took ${metrics.totalDurationMs}ms`);
    }

    if metrics.llmCallCount > 5 {
        log:printWarn(string `HIGH_LLM_CALLS: Agent ${metrics.agentId} made ${metrics.llmCallCount} LLM calls in one run`);
    }
}
```

:::tip
Start with generous alert thresholds and tighten them as you learn your integration's normal operating parameters. False alerts lead to alert fatigue.
:::

## Dashboard Recommendations

Build a dashboard with these panels:

1. **Request volume** -- Requests per minute, broken down by agent
2. **Latency distribution** -- P50, P95, P99 latency for LLM calls
3. **Token consumption** -- Tokens per hour, trending over time
4. **Error rate** -- Percentage of failed calls, by error type
5. **Cost tracker** -- Estimated daily/weekly/monthly spend
6. **Agent effectiveness** -- Success rate per agent type

## What's Next

- [Debug Agent Behavior](debugging-agent-behavior.md) -- Use metrics to identify and fix agent issues
- [Trace Agent Execution](agent-tracing.md) -- Drill into individual agent runs
- [Manage Tokens and Costs](/docs/genai/guardrails/token-cost-management) -- Act on token consumption data
