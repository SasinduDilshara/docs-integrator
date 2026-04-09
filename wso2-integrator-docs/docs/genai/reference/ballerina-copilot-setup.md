---
title: "Ballerina Copilot Setup and Usage Guide"
description: "Set up and configure the Ballerina AI Copilot, including authentication, data flow, BYOK options, and transparency practices."
---

# Ballerina Copilot Setup and Usage Guide

WSO2 Integrator includes an AI-powered Copilot that assists you while building integrations. This page explains exactly how your data is handled, what security measures are in place, and what practices you should follow to use AI features responsibly. Transparency is a core principle -- you should always know where your data goes and how long it stays there.

## AI Copilot Macro Architecture

The AI Copilot consists of several layers, each with a clear boundary:

1. **AI Copilot Code** -- a VS Code extension running on your local machine that provides in-editor assistance such as code completion, explanations, and suggestions.
2. **Language Server** -- powers intelligent features inside the IDE, including syntax awareness and integration with Copilot services. Processes requests locally and forwards AI-related calls.
3. **BI Intelligence Endpoint** -- a lightweight intermediary service that routes requests to the LLM provider. This layer performs **no data retention**.
4. **LLM Provider** -- Anthropic (directly) or Amazon Bedrock, depending on your configuration.

:::note
The BI Intelligence Endpoint is intentionally lightweight. It authenticates the request, forwards it to the configured LLM provider, and returns the response. No prompts, code snippets, or responses are stored at this layer.
:::

## Authentication via Asgardeo

All AI Copilot requests require authentication to maintain security:

- Users must **log in** before using AI features.
- **Social login** options are supported for ease of use.
- Authentication and session management are handled by **[Asgardeo](https://wso2.com/asgardeo/)**, WSO2's identity provider.

This ensures that only authorized users in your organization can access AI features.

:::info
Asgardeo handles session tokens, so you do not need to manage API keys when using the default WSO2 model provider. Tokens are session-scoped and expire automatically.
:::

## Data Flow and Zero-Retention Policy

Understanding how data moves through the system is essential for compliance and trust. The three key principles are:

### Direct Forwarding to the LLM Provider

Prompts and code context are forwarded directly from the BI Intelligence Endpoint to the configured LLM provider (Anthropic or Bedrock). There is no intermediate storage, caching, or logging of prompt content.

### No Local Storage at BI Intelligence

The BI Intelligence Endpoint does not write prompts, responses, or any user data to disk, database, or any persistent store. It operates as a pure pass-through.

### Real-Time Processing Only

All processing happens in real time. Once the LLM response is returned to the client, no residual data remains at the intermediary layer.

## Bring Your Own Key (BYOK)

Organizations can configure integrations to run using their own model provider accounts. This ensures enterprise-level control over data governance and billing.

### Anthropic Deployment

Connect directly to Anthropic's public deployments using your own API key. Data flows directly between your environment and Anthropic without WSO2 retaining it.

```toml
[ai]
provider = "anthropic"
apiKey = "${ANTHROPIC_API_KEY}"
# Requests go directly to Anthropic, bypassing the WSO2 intermediary
```

### Amazon Bedrock Deployment

Run Claude models on your own Amazon Bedrock environment. This gives you full control over the cloud region, VPC configuration, and billing. Requires an active Claude deployment in your Amazon Bedrock environment.

```toml
[ai]
provider = "bedrock"
region = "${AWS_REGION}"
accessKeyId = "${AWS_ACCESS_KEY_ID}"
secretAccessKey = "${AWS_SECRET_ACCESS_KEY}"
```

:::warning
When using BYOK, you are subject to the provider's data retention and privacy policies. Review your provider's terms to ensure compliance with your organization's data governance requirements.
:::

## Open-Source Copilot Code

The Ballerina AI Copilot code is **open source** and fully auditable. This means:

- The full source code is available for inspection, download, and modification.
- Security teams can validate the behavior of the Copilot and review what data is sent to the LLM provider.
- Enterprises can extend or customize the Copilot to meet internal compliance requirements.

:::tip
If your organization has specific compliance needs, consider forking the Copilot extension and adding custom guardrails -- such as stripping sensitive patterns from prompts before they are sent to the LLM.
:::

All operations are subject to the [Anthropic Data Usage Policy](https://privacy.anthropic.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data) or the chosen model provider's terms.

## Feedback Data Handling

WSO2 collects feedback data only when you explicitly choose to provide it:

**Retention period:**
- Feedback data (such as thumbs up/down ratings) is retained for **1 week only**.
- After 1 week, feedback records are **permanently deleted**.

**Collection scope:**
- Feedback is collected **only when a user explicitly provides it** through the Copilot interface.
- No hidden or passive data collection is performed.

**Transparency:**
- The feedback interface clearly explains what is being collected and why.
- Users always have control over whether to provide feedback.

:::note
Feedback helps improve the AI experience, but it is entirely optional. If you never click the feedback buttons, no feedback data is collected or transmitted.
:::

## Transparency: Disclosing AI Use

Always inform end users when they are interacting with AI-generated content. Include metadata in your API responses that clearly marks content as AI-generated:

```ballerina
import ballerina/ai;
import ballerina/http;

type AIResponse record {|
    string content;
    boolean aiGenerated;
    string model;
    string disclaimer;
|};

final ai:Client aiClient = check new ({model: "gpt-4o-mini"});

service /api on new http:Listener(8080) {

    resource function post query(@http:Payload QueryRequest req) returns AIResponse|error {
        string result = check aiClient->generate(req.question, string);

        return {
            content: result,
            aiGenerated: true,
            model: "gpt-4o-mini",
            disclaimer: "This response was generated by an AI model and may contain inaccuracies. Please verify critical information."
        };
    }
}

type QueryRequest record {|
    string question;
|};
```

Adding an `aiGenerated` flag and a `disclaimer` field to your responses builds trust and meets emerging regulatory requirements around AI transparency.

## Data Retention Summary

| Data Type | Retention Period | Notes |
|---|---|---|
| Code Prompts and Responses | Not stored by BI Intelligence | Forwarded directly to Anthropic or Bedrock |
| User Feedback | 1 week | Retained only when explicitly provided by the user |
| Authentication Tokens | Session-based | Managed securely by Asgardeo |
| Organizational Data | Not stored | Zero-retention policy at BI Intelligence |

## Security Recommendations

Follow these practices for secure AI-powered integrations:

1. **Rotate API keys regularly** -- Use WSO2 Integrator's secrets management for key rotation.
2. **Limit model access** -- Grant LLM access only to integrations and users that need it.
3. **Monitor for anomalies** -- Watch for unusual token usage spikes or unexpected request patterns.
4. **Encrypt data in transit** -- All WSO2 Intelligence Endpoint communication uses TLS.
5. **Review prompts periodically** -- Audit system prompts for information leakage risks.

## Best Practices for Organizations

To ensure maximum security and privacy, avoid sending organizational-specific details such as:

- Customer personal information (names, emails, phone numbers, national IDs)
- Passwords or authentication credentials
- Proprietary business data
- Sensitive internal communications

General best practices:

- **Review all AI-generated code** before implementation -- AI output may contain bugs, security vulnerabilities, or logic errors.
- **Be mindful of what information you include in prompts** -- use generic examples rather than real production data.
- **Follow your organization's data governance policies** -- check with your security team before enabling AI features if your organization has specific data handling policies.

:::warning
AI-generated code is a starting point, not a finished product. Always review suggestions for correctness, security, and alignment with your coding standards before committing them to your codebase.
:::
