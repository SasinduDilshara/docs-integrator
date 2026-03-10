---
sidebar_position: 1
title: "Add Input and Output Guardrails"
description: "Validate and sanitize data before sending to LLMs and after receiving responses to ensure safe, reliable integrations."
---

# Add Input and Output Guardrails

Guardrails protect your integrations from bad input reaching the LLM and bad output reaching your users or downstream systems. You validate input to prevent prompt injection, enforce constraints, and ensure data quality. You validate output to catch hallucinations, enforce schemas, and provide fallback responses when the LLM produces unexpected results.

## Input Guardrails

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

## Output Guardrails

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

## Combining Input and Output Guardrails

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

## What's Next

- [Filter Content](content-filtering.md) -- Block harmful content and redact sensitive data
- [Manage Tokens and Costs](token-cost-management.md) -- Control LLM spending
- [Prompt Engineering for Integrations](/docs/genai/llm-connectivity/prompt-engineering) -- Write prompts that produce guardrail-compliant output
