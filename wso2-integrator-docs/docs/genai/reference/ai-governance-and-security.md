---
title: "AI Governance and Security"
description: "Comprehensive guide to input/output guardrails, content filtering, token and cost management, and responsible AI practices."
---

# AI Governance and Security

AI governance ensures that your AI-powered integrations are safe, compliant, cost-effective, and fair. This page consolidates all governance and security concerns into a single reference: input/output guardrails, content filtering, token and cost management, and responsible AI practices.

## Input/Output Guardrails

Guardrails protect your integrations from bad input reaching the LLM and bad output reaching your users or downstream systems. You validate input to prevent prompt injection, enforce constraints, and ensure data quality. You validate output to catch hallucinations, enforce schemas, and provide fallback responses when the LLM produces unexpected results.

### Length and Format Validation

Check that input meets basic requirements before incurring LLM costs:

```ballerina
import ballerina/log;

type ValidationResult record {|
    boolean valid;
    string? reason;
|};

function validateInput(string userInput) returns ValidationResult {
    // Reject empty input
    if userInput.trim().length() == 0 {
        return {valid: false, reason: "Input cannot be empty"};
    }

    // Enforce maximum length
    if userInput.length() > 10000 {
        return {valid: false, reason: "Input exceeds maximum length of 10,000 characters"};
    }

    // Check for suspicious patterns (basic prompt injection defense)
    string[] blockedPatterns = [
        "ignore previous instructions",
        "ignore all instructions",
        "system prompt:",
        "you are now"
    ];

    string lowerInput = userInput.toLowerAscii();
    foreach string pattern in blockedPatterns {
        if lowerInput.includes(pattern) {
            log:printWarn(string `Blocked input containing suspicious pattern: ${pattern}`);
            return {valid: false, reason: "Input contains blocked content"};
        }
    }

    return {valid: true, reason: ()};
}
```

### Schema Validation for Structured Input

When your integration accepts structured data, validate it before constructing prompts:

```ballerina
type CustomerQuery record {|
    string customerId;
    string question;
    string? context;
|};

function validateCustomerQuery(CustomerQuery query) returns ValidationResult {
    if query.customerId.length() == 0 {
        return {valid: false, reason: "Customer ID is required"};
    }

    if query.question.length() < 10 {
        return {valid: false, reason: "Question is too short to be meaningful"};
    }

    return {valid: true, reason: ()};
}
```

:::warning
Never trust user input. Even if your service is internal, validate all input before including it in LLM prompts. Prompt injection attacks can cause the LLM to ignore instructions, leak system prompts, or produce harmful output.
:::

### Input Sanitization

Strip or escape content that could interfere with prompt structure:

```ballerina
function sanitizeInput(string rawInput) returns string {
    string sanitized = rawInput;

    // Remove control characters
    sanitized = sanitized.trim();

    // Escape delimiter tokens used in your prompts
    sanitized = re `<\/?[A-Z_]+>`.replaceAll(sanitized, "[REMOVED]");

    return sanitized;
}
```

### Type-Safe Output Validation

WSO2 Integrator's direct LLM invocation automatically validates output against your Ballerina record types. If the LLM response does not conform to the type, the call returns an error:

```ballerina
import ballerinax/ai;

type ProductRecommendation record {|
    string productName;
    string reason;
    decimal price;
    float relevanceScore;
|};

function getRecommendation(string query) returns ProductRecommendation|error {
    // Type conformance is enforced automatically
    ProductRecommendation result = check aiClient->generate(
        string `Recommend a product for: ${query}`,
        ProductRecommendation
    );

    // Additional business logic validation
    if result.price < 0d {
        return error("Invalid recommendation: negative price");
    }
    if result.relevanceScore < 0.0 || result.relevanceScore > 1.0 {
        return error("Invalid recommendation: relevance score out of range");
    }

    return result;
}
```

### Fallback Responses

When output validation fails, provide a safe default rather than propagating errors:

```ballerina
function getRecommendationWithFallback(string query) returns ProductRecommendation {
    ProductRecommendation|error result = getRecommendation(query);

    if result is error {
        // Return a safe default
        return {
            productName: "Unable to generate recommendation",
            reason: "Please try again or contact support",
            price: 0d,
            relevanceScore: 0.0
        };
    }

    return result;
}
```

:::tip
Log guardrail violations so you can identify patterns. Frequent violations on the same prompt may indicate a prompt design issue rather than LLM unreliability.
:::

### Content Safety Checks

Validate that LLM output does not contain inappropriate or harmful content:

```ballerina
function checkOutputSafety(string llmOutput) returns ValidationResult {
    // Check for refusal patterns
    string[] refusalIndicators = [
        "I cannot",
        "I'm unable to",
        "As an AI",
        "I apologize, but"
    ];

    foreach string indicator in refusalIndicators {
        if llmOutput.includes(indicator) {
            return {valid: false, reason: "LLM refused to answer"};
        }
    }

    // Check output is not suspiciously short
    if llmOutput.length() < 10 {
        return {valid: false, reason: "Output is too short"};
    }

    return {valid: true, reason: ()};
}
```

### Combining Input and Output Guardrails

Build a reusable guardrail pipeline for your integration:

```ballerina
import ballerinax/ai;
import ballerina/log;

final ai:Client aiClient = check new ({model: "gpt-4o-mini"});

function processWithGuardrails(string userInput) returns string|error {
    // Step 1: Input validation
    ValidationResult inputCheck = validateInput(userInput);
    if !inputCheck.valid {
        return error(string `Input rejected: ${inputCheck.reason ?: "unknown"}`);
    }

    // Step 2: Sanitize
    string sanitized = sanitizeInput(userInput);

    // Step 3: LLM call
    string llmOutput = check aiClient->generate(sanitized, string);

    // Step 4: Output validation
    ValidationResult outputCheck = checkOutputSafety(llmOutput);
    if !outputCheck.valid {
        log:printWarn(string `Output guardrail triggered: ${outputCheck.reason ?: "unknown"}`);
        return "I'm sorry, I couldn't process that request. Please try rephrasing.";
    }

    return llmOutput;
}
```

## Content Filtering

Content filtering prevents harmful, sensitive, or off-topic content from flowing through your integrations. You apply filters on both the input side (before the LLM sees user data) and the output side (before results reach users or downstream systems).

### PII Detection and Redaction

Personally identifiable information should never reach an LLM unless absolutely necessary. Redact PII from input before sending it to the model.

```ballerina
import ballerina/regex;
import ballerina/log;

type PiiResult record {|
    string redactedText;
    string[] detectedTypes;
|};

function redactPii(string input) returns PiiResult {
    string redacted = input;
    string[] detected = [];

    // Email addresses
    if re `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`.isFullMatch(input) {
        // Pattern found
    }
    redacted = re `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`.replaceAll(redacted, "[EMAIL_REDACTED]");
    if redacted != input {
        detected.push("email");
    }

    // Phone numbers (US format)
    string afterPhone = re `(\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}`.replaceAll(redacted, "[PHONE_REDACTED]");
    if afterPhone != redacted {
        detected.push("phone");
        redacted = afterPhone;
    }

    // Social Security Numbers
    string afterSsn = re `\d{3}-\d{2}-\d{4}`.replaceAll(redacted, "[SSN_REDACTED]");
    if afterSsn != redacted {
        detected.push("ssn");
        redacted = afterSsn;
    }

    // Credit card numbers
    string afterCc = re `\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}`.replaceAll(redacted, "[CC_REDACTED]");
    if afterCc != redacted {
        detected.push("credit_card");
        redacted = afterCc;
    }

    if detected.length() > 0 {
        log:printInfo(string `Redacted PII types: ${detected.toString()}`);
    }

    return {redactedText: redacted, detectedTypes: detected};
}
```

:::warning
Regex-based PII detection catches common patterns but is not exhaustive. For production systems handling sensitive data, consider using a dedicated PII detection service alongside regex patterns.
:::

### Topic Restriction

Limit your integration to specific topics so the LLM stays focused on its intended purpose:

```ballerina
import ballerinax/ai;

type TopicCheck record {|
    boolean onTopic;
    string detectedTopic;
|};

final ai:Client filterClient = check new ({model: "gpt-4o-mini"});

function checkTopic(string userMessage, string[] allowedTopics) returns TopicCheck|error {
    string topicList = string:'join(", ", ...allowedTopics);

    return filterClient->generate(
        string `Determine if the following message is about one of these allowed topics: ${topicList}.
                Message: "${userMessage}"
                If the message is on-topic, return the matching topic. If off-topic, return "off-topic".`,
        TopicCheck
    );
}

function enforceTopicRestriction(string userMessage) returns string|error {
    string[] allowedTopics = ["product support", "billing", "account management"];

    TopicCheck topicResult = check checkTopic(userMessage, allowedTopics);

    if !topicResult.onTopic {
        return "I can only help with product support, billing, and account management questions.";
    }

    // Proceed with the main LLM call
    return filterClient->generate(userMessage, string);
}
```

### Custom Filter Rules

Define organization-specific rules that block or transform content:

```ballerina
type FilterRule record {|
    string name;
    string pattern;
    string action;  // "block", "redact", "warn"
    string replacement?;
|};

function applyCustomFilters(string content, FilterRule[] rules) returns string|error {
    string filtered = content;

    foreach FilterRule rule in rules {
        boolean matches = re `${rule.pattern}`.find(filtered) is regex:Span;

        if matches {
            match rule.action {
                "block" => {
                    return error(string `Content blocked by rule: ${rule.name}`);
                }
                "redact" => {
                    string replacement = rule.replacement ?: "[REDACTED]";
                    filtered = re `${rule.pattern}`.replaceAll(filtered, replacement);
                }
                "warn" => {
                    log:printWarn(string `Filter rule triggered: ${rule.name}`);
                }
            }
        }
    }

    return filtered;
}
```

Configure rules in your `Config.toml`:

```toml
[[filterRules]]
name = "competitor_mentions"
pattern = "(?i)(competitor-a|competitor-b)"
action = "redact"
replacement = "[COMPETITOR]"

[[filterRules]]
name = "internal_urls"
pattern = "https?://internal\\..+"
action = "block"
```

:::tip
Keep filter rules configurable rather than hard-coded. This lets your operations team update rules without redeploying the integration.
:::

### Harmful Content Blocking

Use the LLM itself as a content classifier to catch harmful output:

```ballerina
type SafetyAssessment record {|
    boolean safe;
    string? category;
    string? explanation;
|};

function assessContentSafety(string content) returns SafetyAssessment|error {
    return filterClient->generate(
        string `Assess the following content for safety.
                Check for: hate speech, violence, illegal activity, self-harm, explicit content.
                Content: "${content}"`,
        SafetyAssessment
    );
}

function filterOutput(string llmOutput) returns string|error {
    SafetyAssessment safety = check assessContentSafety(llmOutput);

    if !safety.safe {
        log:printWarn(string `Unsafe content blocked. Category: ${safety.category ?: "unknown"}`);
        return "I'm unable to provide that response. Please try a different question.";
    }

    return llmOutput;
}
```

### Building a Filter Pipeline

Combine multiple filters into a single pipeline:

```ballerina
function filterPipeline(string input) returns string|error {
    // Step 1: PII redaction
    PiiResult piiResult = redactPii(input);
    string filtered = piiResult.redactedText;

    // Step 2: Topic check
    TopicCheck topicCheck = check checkTopic(filtered, ["support", "billing"]);
    if !topicCheck.onTopic {
        return error("Off-topic input rejected");
    }

    // Step 3: Custom rules
    filtered = check applyCustomFilters(filtered, getConfiguredRules());

    return filtered;
}
```

## Token and Cost Management

LLM costs scale directly with token usage, and unmonitored integrations can generate surprising bills. You need visibility into how many tokens each request, agent, and integration consumes, along with controls that prevent runaway spending.

### Understanding Token Costs

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

### Tracking Token Usage Per Request

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

### Setting Cost Limits

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

### Budget Alerts

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

### Per-Agent Cost Tracking

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

### Optimizing Prompts for Cost

Reduce token usage without sacrificing quality.

#### Technique 1: Concise System Prompts

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

#### Technique 2: Use Smaller Models for Simple Tasks

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

#### Technique 3: Cache Repeated Queries

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

## Responsible AI

LLMs can reflect and amplify biases from their training data. In integration workflows, this matters most when AI influences decisions about people. This section covers bias mitigation, human-in-the-loop patterns, and audit trails.

### Bias Detection in Output

```ballerina
type BiasCheckResult record {|
    boolean potentialBias;
    string[] flaggedTerms;
    string recommendation;
|};

function checkForBias(string llmOutput, string context) returns BiasCheckResult|error {
    return aiClient->generate(
        string `Review the following AI-generated output for potential bias related to
                gender, race, age, disability, or other protected characteristics.
                Context: ${context}
                Output to review: "${llmOutput}"`,
        BiasCheckResult
    );
}
```

### Balanced Prompt Design

Structure prompts to reduce bias:

```ballerina
// Instead of open-ended generation that may reflect biases
string biasedPrompt = "Describe the ideal candidate for this role.";

// Use structured, criteria-based prompts
string balancedPrompt = string `
    Evaluate the candidate against these specific, measurable criteria only:
    1. Years of relevant experience
    2. Required technical certifications
    3. Demonstrated project outcomes

    Do not consider or mention: name, gender, age, ethnicity, or any
    demographic information. Focus solely on qualifications.
`;
```

### Human-in-the-Loop Patterns

For high-stakes decisions, route AI outputs through human review before acting:

```ballerina
import ballerinax/ai;
import ballerina/http;

type ApprovalRequest record {|
    string id;
    string aiRecommendation;
    float confidence;
    string originalInput;
    string status;  // "pending", "approved", "rejected"
|};

// Confidence threshold for auto-approval
configurable float autoApproveThreshold = 0.95;

function processWithHumanReview(string input) returns ApprovalRequest|error {
    type RecommendationResult record {|
        string recommendation;
        float confidence;
    |};

    RecommendationResult result = check aiClient->generate(
        string `Analyze and provide a recommendation: ${input}`,
        RecommendationResult
    );

    string status = result.confidence >= autoApproveThreshold ? "approved" : "pending";

    ApprovalRequest request = {
        id: generateRequestId(),
        aiRecommendation: result.recommendation,
        confidence: result.confidence,
        originalInput: input,
        status: status
    };

    if status == "pending" {
        // Queue for human review
        check queueForReview(request);
    }

    return request;
}

function generateRequestId() returns string {
    return "req-" + checkpanic int:toHexString(checkpanic int:fromString(
        string `${time:utcNow()[0]}`
    ));
}
```

:::tip
Set your auto-approval threshold conservatively at first (e.g., 0.95) and adjust downward as you build confidence in the model's accuracy for your specific use case.
:::

### Audit Trails

Log every AI decision with enough context to reconstruct the reasoning:

```ballerina
import ballerina/log;
import ballerina/time;

type AuditEntry record {|
    string timestamp;
    string requestId;
    string model;
    string inputSummary;
    string outputSummary;
    int tokensUsed;
    decimal cost;
    string decision;
    string? humanReviewerId;
|};

function logAuditEntry(AuditEntry entry) {
    log:printInfo(string `AI_AUDIT | ${entry.timestamp} | ${entry.requestId} | ` +
        string `model=${entry.model} | tokens=${entry.tokensUsed} | ` +
        string `cost=$${entry.cost} | decision=${entry.decision} | ` +
        string `reviewer=${entry.humanReviewerId ?: "auto"}`);
}

function createAuditEntry(string requestId, string model, string input,
        string output, int tokens, decimal cost) returns AuditEntry {
    return {
        timestamp: time:utcToString(time:utcNow()),
        requestId: requestId,
        model: model,
        inputSummary: input.length() > 200 ? input.substring(0, 200) + "..." : input,
        outputSummary: output.length() > 200 ? output.substring(0, 200) + "..." : output,
        tokensUsed: tokens,
        cost: cost,
        decision: output,
        humanReviewerId: ()
    };
}
```

### Feedback Collection

```ballerina
type FeedbackEntry record {|
    string responseId;
    string rating;       // "helpful", "not_helpful", "harmful"
    string? comment;
    string timestamp;
|};

function recordFeedback(FeedbackEntry feedback) returns error? {
    log:printInfo(string `FEEDBACK | ${feedback.responseId} | ${feedback.rating} | ${feedback.comment ?: "none"}`);
    // Persist to database for analysis
}
```
