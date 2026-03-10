---
sidebar_position: 3
title: "Manage Tokens and Costs"
description: "Track token usage, set cost limits, configure budget alerts, and optimize prompts for cost-efficient LLM integrations."
---

# Manage Tokens and Costs

LLM costs scale directly with token usage, and unmonitored integrations can generate surprising bills. You need visibility into how many tokens each request, agent, and integration consumes, along with controls that prevent runaway spending. WSO2 Integrator provides the building blocks for comprehensive cost management.

## Understanding Token Costs

Every LLM call consumes tokens for both input (prompt) and output (completion). Pricing varies significantly across models:

| Model | Input Cost (per 1M tokens) | Output Cost (per 1M tokens) |
|-------|---------------------------|----------------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude Sonnet | $3.00 | $15.00 |
| Claude Haiku | $0.25 | $1.25 |
| Gemini Flash | $0.075 | $0.30 |

:::info
When using the WSO2 Intelligence Endpoint, token costs are included in your WSO2 subscription. The tracking recommendations below still apply for monitoring usage and optimizing performance.
:::

## Tracking Token Usage Per Request

Capture token metrics on every LLM call:

```ballerina
import ballerinax/ai;
import ballerina/log;

type TokenUsage record {|
    int promptTokens;
    int completionTokens;
    int totalTokens;
    decimal estimatedCost;
|};

final ai:Client aiClient = check new ({model: "gpt-4o-mini"});

function generateWithTracking(string prompt) returns [string, TokenUsage]|error {
    ai:GenerationResult result = check aiClient->generateWithMetadata(prompt, string);

    decimal inputCost = <decimal>result.usage.promptTokens / 1000000d * 0.15d;
    decimal outputCost = <decimal>result.usage.completionTokens / 1000000d * 0.60d;

    TokenUsage usage = {
        promptTokens: result.usage.promptTokens,
        completionTokens: result.usage.completionTokens,
        totalTokens: result.usage.promptTokens + result.usage.completionTokens,
        estimatedCost: inputCost + outputCost
    };

    log:printInfo(string `Tokens used: ${usage.totalTokens}, Cost: $${usage.estimatedCost}`);

    return [result.content, usage];
}
```

## Setting Cost Limits

Prevent individual requests or sessions from exceeding a budget:

```ballerina
import ballerina/log;

configurable decimal maxCostPerRequest = 0.10d;
configurable decimal maxCostPerDay = 50.00d;

// In-memory daily tracker (use a persistent store in production)
isolated decimal dailySpend = 0d;

function checkBudget(decimal estimatedCost) returns error? {
    if estimatedCost > maxCostPerRequest {
        return error(string `Request exceeds per-request limit: $${estimatedCost} > $${maxCostPerRequest}`);
    }

    lock {
        if dailySpend + estimatedCost > maxCostPerDay {
            return error(string `Daily budget exhausted: $${dailySpend} spent of $${maxCostPerDay} limit`);
        }
    }
}

function recordSpend(decimal cost) {
    lock {
        dailySpend += cost;
    }
    log:printInfo(string `Daily spend: $${dailySpend}`);
}
```

:::warning
In-memory tracking resets on service restart. For production deployments, persist usage data to a database or use WSO2 Integrator's built-in metrics. See the [Deploy and Operate](/docs/deploy-operate) section for persistent storage options.
:::

## Budget Alerts

Send notifications when spending approaches thresholds:

```ballerina
import ballerina/email;
import ballerina/log;

configurable decimal alertThreshold = 0.80d;  // Alert at 80% of daily budget

function checkAndAlert(decimal currentSpend, decimal dailyBudget) returns error? {
    decimal utilization = currentSpend / dailyBudget;

    if utilization >= 1.0d {
        log:printError("CRITICAL: Daily LLM budget exhausted");
        check sendAlert("CRITICAL", string `LLM budget exhausted: $${currentSpend}/$${dailyBudget}`);
    } else if utilization >= alertThreshold {
        log:printWarn(string `Budget warning: ${(utilization * 100d).toString()}% used`);
        check sendAlert("WARNING", string `LLM budget at ${(utilization * 100d).toString()}%`);
    }
}

function sendAlert(string severity, string message) returns error? {
    // Send via your preferred channel: email, Slack, PagerDuty, etc.
    log:printInfo(string `[${severity}] ${message}`);
}
```

## Per-Agent Cost Tracking

When running multiple agents, track costs individually:

```ballerina
type AgentCostRecord record {|
    string agentId;
    int totalRequests;
    int totalTokens;
    decimal totalCost;
|};

map<AgentCostRecord> agentCosts = {};

function trackAgentCost(string agentId, TokenUsage usage) {
    AgentCostRecord current = agentCosts[agentId] ?: {
        agentId: agentId,
        totalRequests: 0,
        totalTokens: 0,
        totalCost: 0d
    };

    current.totalRequests += 1;
    current.totalTokens += usage.totalTokens;
    current.totalCost += usage.estimatedCost;

    agentCosts[agentId] = current;
}

function getAgentCostReport() returns AgentCostRecord[] {
    return agentCosts.toArray();
}
```

## Optimizing Prompts for Cost

Reduce token usage without sacrificing quality.

### Technique 1: Concise System Prompts

```ballerina
// Expensive: verbose system prompt
string verbosePrompt = string `
    You are a highly skilled and experienced data extraction specialist
    who has been trained to carefully and accurately extract structured
    information from various types of documents...
`;  // ~40 tokens

// Optimized: same instructions, fewer tokens
string concisePrompt = string `
    Extract structured data from the input. Return only requested fields.
    Use null for missing values.
`;  // ~18 tokens
```

### Technique 2: Use Smaller Models for Simple Tasks

```ballerina
import ballerinax/ai;

// Route requests to the appropriate model based on complexity
function routeByComplexity(string task, string input) returns string|error {
    ai:Client miniClient = check new ({model: "gpt-4o-mini"});
    ai:Client fullClient = check new ({model: "gpt-4o"});

    match task {
        "classify"|"extract" => {
            return miniClient->generate(input, string);
        }
        "analyze"|"reason" => {
            return fullClient->generate(input, string);
        }
        _ => {
            return miniClient->generate(input, string);
        }
    }
}
```

### Technique 3: Cache Repeated Queries

```ballerina
import ballerina/cache;

final cache:Cache responseCache = new ({
    capacity: 1000,
    evictionFactor: 0.2,
    defaultMaxAge: 3600  // 1 hour
});

function generateWithCache(string prompt) returns string|error {
    string cacheKey = prompt.toBytes().toBase64();

    any|error cached = responseCache.get(cacheKey);
    if cached is string {
        log:printInfo("Cache hit - zero tokens used");
        return cached;
    }

    string result = check aiClient->generate(prompt, string);
    check responseCache.put(cacheKey, result);
    return result;
}
```

:::tip
Caching works best for classification and extraction tasks where the same input always produces the same output. Avoid caching conversational or creative outputs.
:::

## What's Next

- [Responsible AI Practices](responsible-ai.md) -- Broader governance for AI-powered integrations
- [Monitor Performance Metrics](/docs/genai/observability/performance-metrics) -- Track token usage alongside latency metrics
- [Model Selection Guide](/docs/genai/llm-connectivity/model-selection) -- Choose cost-effective models
