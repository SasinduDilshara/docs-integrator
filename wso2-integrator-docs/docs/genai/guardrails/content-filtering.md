---
sidebar_position: 2
title: "Filter Content"
description: "Block harmful content, detect and redact PII, restrict topics, and apply custom filtering rules in LLM-powered integrations."
---

# Filter Content

Content filtering prevents harmful, sensitive, or off-topic content from flowing through your integrations. You apply filters on both the input side (before the LLM sees user data) and the output side (before results reach users or downstream systems). WSO2 Integrator gives you the tools to build these filters directly into your integration logic.

## PII Detection and Redaction

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

## Topic Restriction

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

## Custom Filter Rules

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

## Harmful Content Blocking

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

## Building a Filter Pipeline

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

## What's Next

- [Manage Tokens and Costs](token-cost-management.md) -- Control spending on filter and main LLM calls
- [Responsible AI Practices](responsible-ai.md) -- Broader AI safety and transparency guidelines
- [Add Input and Output Guardrails](input-output-guardrails.md) -- Schema-level validation
