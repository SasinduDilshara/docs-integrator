---
sidebar_position: 2
title: "Log Agent Conversations"
description: "Capture and structure agent conversation history with proper retention policies and privacy safeguards."
---

# Log Agent Conversations

Conversation logs are your primary record of what agents said to users and why. They serve debugging, compliance, quality assurance, and continuous improvement. WSO2 Integrator provides structured logging that captures the full context of each agent interaction while respecting privacy requirements.

## Structured Log Format

Use a consistent structure for all agent conversation logs:

```ballerina
import ballerina/log;
import ballerina/time;

type ConversationLogEntry record {|
    string conversationId;
    string turnId;
    int turnNumber;
    string timestamp;
    string role;            // "user", "agent", "system", "tool"
    string content;
    string? model;
    int? tokensUsed;
    string? toolName;
    map<string>? metadata;
|};

function logConversationTurn(ConversationLogEntry entry) {
    // Structured JSON log for machine parsing
    log:printInfo(string `CONVERSATION_LOG`,
        conversationId = entry.conversationId,
        turnId = entry.turnId,
        turnNumber = entry.turnNumber,
        role = entry.role,
        contentLength = entry.content.length(),
        model = entry.model ?: "n/a",
        tokensUsed = entry.tokensUsed ?: 0
    );
}
```

## Logging a Full Conversation

Wrap your agent interactions with logging at each turn:

```ballerina
import ballerinax/ai;
import ballerina/uuid;

final ai:Client aiClient = check new ({model: "gpt-4o-mini"});

type Message record {|
    string role;
    string content;
|};

class ConversationLogger {
    private string conversationId;
    private int turnCount = 0;
    private ConversationLogEntry[] entries = [];

    function init() {
        self.conversationId = uuid:createType4AsString();
    }

    function logUserMessage(string message) {
        self.turnCount += 1;
        ConversationLogEntry entry = {
            conversationId: self.conversationId,
            turnId: uuid:createType4AsString(),
            turnNumber: self.turnCount,
            timestamp: time:utcToString(time:utcNow()),
            role: "user",
            content: message,
            model: (),
            tokensUsed: (),
            toolName: (),
            metadata: ()
        };
        self.entries.push(entry);
        logConversationTurn(entry);
    }

    function logAgentResponse(string response, string model, int tokens) {
        ConversationLogEntry entry = {
            conversationId: self.conversationId,
            turnId: uuid:createType4AsString(),
            turnNumber: self.turnCount,
            timestamp: time:utcToString(time:utcNow()),
            role: "agent",
            content: response,
            model: model,
            tokensUsed: tokens,
            toolName: (),
            metadata: ()
        };
        self.entries.push(entry);
        logConversationTurn(entry);
    }

    function logToolCall(string toolName, string input, string output) {
        ConversationLogEntry entry = {
            conversationId: self.conversationId,
            turnId: uuid:createType4AsString(),
            turnNumber: self.turnCount,
            timestamp: time:utcToString(time:utcNow()),
            role: "tool",
            content: string `Input: ${input} | Output: ${output}`,
            model: (),
            tokensUsed: (),
            toolName: toolName,
            metadata: ()
        };
        self.entries.push(entry);
        logConversationTurn(entry);
    }

    function getConversationId() returns string {
        return self.conversationId;
    }
}
```

## Using the Logger in a Service

Integrate conversation logging into your agent service:

```ballerina
import ballerina/http;

service /agent on new http:Listener(8080) {

    resource function post chat(@http:Payload ChatRequest req) returns ChatResponse|error {
        ConversationLogger logger = new;

        // Log user input
        logger.logUserMessage(req.message);

        // Get agent response
        string response = check aiClient->generate(req.message, string);

        // Log agent response
        logger.logAgentResponse(response, "gpt-4o-mini", 0);

        return {
            conversationId: logger.getConversationId(),
            response: response
        };
    }
}

type ChatRequest record {|
    string message;
    string? conversationId;
|};

type ChatResponse record {|
    string conversationId;
    string response;
|};
```

## Privacy Considerations

Conversation logs can contain sensitive user data. Apply privacy controls before persisting logs.

```ballerina
configurable boolean redactPiiInLogs = true;
configurable boolean logFullContent = false;

function sanitizeForLogging(string content) returns string {
    if !logFullContent {
        // Log only a truncated version
        if content.length() > 500 {
            return content.substring(0, 500) + "... [truncated]";
        }
    }

    if redactPiiInLogs {
        string sanitized = content;
        sanitized = re `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`.replaceAll(sanitized, "[EMAIL]");
        sanitized = re `\d{3}-\d{2}-\d{4}`.replaceAll(sanitized, "[SSN]");
        sanitized = re `\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}`.replaceAll(sanitized, "[CC]");
        return sanitized;
    }

    return content;
}
```

:::warning
Check your organization's data retention policies and applicable regulations (GDPR, CCPA, HIPAA) before logging conversation content. Some jurisdictions require explicit user consent for storing conversation data.
:::

## Retention Policies

Configure how long conversation logs are kept:

```toml
[conversationLogs]
# How long to retain detailed conversation logs
retentionDays = 90

# How long to retain aggregated statistics
statsRetentionDays = 365

# Whether to archive logs before deletion
archiveBeforeDelete = true
archiveLocation = "s3://logs-archive/conversations/"
```

```ballerina
import ballerina/time;
import ballerina/log;

function enforceRetention(ConversationLogEntry[] entries, int retentionDays) returns ConversationLogEntry[] {
    time:Utc cutoff = time:utcAddSeconds(time:utcNow(), -retentionDays * 86400);
    string cutoffStr = time:utcToString(cutoff);

    return from ConversationLogEntry entry in entries
        where entry.timestamp > cutoffStr
        select entry;
}
```

:::tip
Use log levels strategically: log metadata (conversation ID, turn count, token usage) at INFO level, and full content at DEBUG level. This way production logs stay lean while you can enable detailed logging when troubleshooting.
:::

## Querying Conversation History

Build a simple API to retrieve conversation logs for debugging:

```ballerina
service /admin on new http:Listener(9090) {

    resource function get conversations/[string conversationId]() returns ConversationLogEntry[]|error {
        // Retrieve from your log storage
        return getConversationEntries(conversationId);
    }

    resource function get conversations() returns ConversationSummary[]|error {
        // Return recent conversation summaries
        return getRecentConversations(100);
    }
}

type ConversationSummary record {|
    string conversationId;
    int turnCount;
    string firstTimestamp;
    string lastTimestamp;
    int totalTokens;
|};
```

## What's Next

- [Monitor Performance Metrics](performance-metrics.md) -- Track latency and throughput across conversations
- [Debug Agent Behavior](debugging-agent-behavior.md) -- Use conversation logs to diagnose agent issues
- [Responsible AI Practices](/docs/genai/guardrails/responsible-ai) -- Privacy and compliance for AI systems
