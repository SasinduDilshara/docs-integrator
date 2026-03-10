---
sidebar_position: 5
title: "Bind Tools to Agents"
description: Define tools for agents, connect them to external services, and generate tools from functions, connectors, and OpenAPI specs.
---

# Bind Tools to Agents

Tools give agents the ability to interact with the world beyond conversation. When you bind tools to an agent, the LLM can decide when to call them, what arguments to pass, and how to interpret the results — all automatically.

This guide covers defining tools, connecting them to real services, generating tools from existing integrations, and handling errors. The examples are based on a Personal AI Assistant that manages email and calendar through Gmail and Google Calendar connectors.

## Define a Tool from a Function

The simplest way to create a tool is to annotate a Ballerina function with `@agent:Tool`:

```ballerina
import ballerinax/ai.agent;

@agent:Tool
isolated function listUnreadEmails(int maxResults = 10) returns Email[]|error {
    // The agent calls this when the user asks about unread emails
    gmail:Client gmailClient = check new ({auth: {token: gmailAccessToken}});
    gmail:MessageListPage messages = check gmailClient->listMessages(
        userId = "me",
        q = "is:unread",
        maxResults = maxResults
    );
    return messages.messages.map(toEmail);
}
```

The LLM uses the function name, parameter names, types, and return type to understand what the tool does. You can add a description for more clarity:

```ballerina
@agent:Tool {
    description: "Read a specific email by its ID and return the full content"
}
isolated function readSpecificEmail(string emailId) returns EmailContent|error {
    gmail:Client gmailClient = check new ({auth: {token: gmailAccessToken}});
    gmail:Message message = check gmailClient->readMessage(userId = "me", messageId = emailId);
    return toEmailContent(message);
}
```

:::info
Always use descriptive function names and parameter names. The LLM relies on these to decide when and how to use the tool. A function named `f1` with a parameter `x` gives the LLM no useful information.
:::

## Build a Complete Tool Set

Here is a complete Personal AI Assistant with five tools for email and calendar management:

```ballerina
import ballerinax/ai.agent;
import ballerinax/googleapis.gmail;
import ballerinax/googleapis.calendar;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;
configurable string googleAccessToken = ?;

final gmail:Client gmailClient = check new ({auth: {token: googleAccessToken}});
final calendar:Client calendarClient = check new ({auth: {token: googleAccessToken}});

// --- Email Tools ---

@agent:Tool {description: "List unread emails from the user's inbox"}
isolated function listUnreadEmails(int maxResults = 10) returns record{}[]|error {
    return check gmailClient->listMessages(userId = "me", q = "is:unread", maxResults = maxResults);
}

@agent:Tool {description: "Read a specific email by its message ID"}
isolated function readSpecificEmail(string messageId) returns record{}|error {
    return check gmailClient->readMessage(userId = "me", messageId = messageId);
}

@agent:Tool {description: "Send an email to a recipient"}
isolated function sendEmail(string to, string subject, string body) returns string|error {
    gmail:Message result = check gmailClient->sendMessage(userId = "me", message = {
        to: [to],
        subject: subject,
        bodyInHtml: body
    });
    return string `Email sent successfully. Message ID: ${result.id}`;
}

// --- Calendar Tools ---

@agent:Tool {description: "List calendar events for a given date range"}
isolated function listCalendarEvents(string timeMin, string timeMax) returns record{}[]|error {
    return check calendarClient->getEvents(
        calendarId = "primary",
        timeMin = timeMin,
        timeMax = timeMax
    );
}

@agent:Tool {description: "Create a new calendar event"}
isolated function createCalendarEvent(string summary, string startTime, string endTime,
        string? description = ()) returns string|error {
    calendar:Event event = check calendarClient->createEvent(calendarId = "primary", event = {
        summary: summary,
        description: description,
        'start: {dateTime: startTime},
        end: {dateTime: endTime}
    });
    return string `Event created: ${event.summary ?: "Untitled"} (${event.id})`;
}

// --- Agent ---

final agent:ChatAgent personalAssistant = check new (
    systemPrompt = string `You are a personal AI assistant that manages email and calendar.
        - When listing emails, show sender, subject, and a brief preview.
        - When creating calendar events, confirm the details before creating.
        - Use the user identifier "me" for Gmail and "primary" for calendar.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [listUnreadEmails, readSpecificEmail, sendEmail, listCalendarEvents, createCalendarEvent]
);
```

## Generate Tools from Connector Actions

Instead of wrapping each connector method manually, you can auto-generate tools from connector actions:

```ballerina
import ballerinax/ai.agent;
import ballerinax/googleapis.gmail;

final gmail:Client gmailClient = check new ({auth: {token: googleAccessToken}});

// Auto-generate tools from connector actions
final agent:Tool[] emailTools = check agent:generateTools(gmailClient, [
    "listMessages",
    "readMessage",
    "sendMessage"
]);

final agent:ChatAgent emailAgent = check new (
    systemPrompt = "You manage the user's email.",
    model = myModel,
    tools = emailTools
);
```

## Generate Tools from OpenAPI Specifications

If you have an OpenAPI specification for an external service, you can generate tools directly from it:

```ballerina
import ballerinax/ai.agent;

final agent:Tool[] apiTools = check agent:generateToolsFromOpenApi("./specs/inventory-api.yaml", {
    baseUrl: "https://api.example.com",
    auth: {token: apiToken},
    includePaths: ["/products", "/orders"],
    excludeOperations: ["deleteProduct"]
});

final agent:ChatAgent inventoryAgent = check new (
    systemPrompt = "You help manage product inventory.",
    model = myModel,
    tools = apiTools
);
```

:::tip
Use `includePaths` and `excludeOperations` to limit which API operations become tools. Exposing too many tools can overwhelm the LLM and reduce accuracy.
:::

## Input and Output Schemas

Tools use Ballerina's type system for input validation and output typing. Define record types for structured data:

```ballerina
type EmailFilter record {|
    string query;
    int maxResults = 10;
    boolean unreadOnly = true;
|};

type EmailSummary record {|
    string id;
    string sender;
    string subject;
    string preview;
    boolean isRead;
|};

@agent:Tool
isolated function searchEmails(EmailFilter filter) returns EmailSummary[]|error {
    // Type-safe inputs and outputs
    return fetchEmails(filter);
}
```

## Handle Tool Errors

When a tool returns an error, the agent receives the error message and can decide how to proceed — retry, try a different approach, or inform the user:

```ballerina
@agent:Tool
isolated function lookupOrder(string orderId) returns record{}|error {
    record{}|error result = orderDb->getOrder(orderId);
    if result is error {
        // Return a descriptive error the agent can reason about
        return error(string `Order ${orderId} not found. The order ID should be in format ORD-XXXXX.`);
    }
    return result;
}
```

:::warning
Avoid throwing unhandled errors from tools. Always return an `error` value with a clear message so the agent can communicate the issue to the user gracefully.
:::

## Parallel Tool Execution

When the LLM decides to call multiple tools in a single reasoning step, WSO2 Integrator executes them in parallel by default. This reduces latency for agents that need data from multiple sources simultaneously.

```ballerina
// If the user asks "What's the weather and do I have meetings today?",
// the agent may call both tools in parallel:
tools = [getWeather, listCalendarEvents]
```

You can disable parallel execution if tools have dependencies:

```ballerina
final agent:ChatAgent assistant = check new (
    systemPrompt = "...",
    model = myModel,
    tools = [tool1, tool2],
    parallelToolExecution = false
);
```

## What's Next

- [Orchestrate Multiple Agents](/docs/genai/agents/multi-agent-orchestration) — Route between specialized agents with different tool sets
- [Use Natural Functions](/docs/genai/agents/natural-functions) — Let the LLM execute logic described in plain language
- [Agent Architecture & Concepts](/docs/genai/agents/architecture-concepts) — Understand how tool calling fits into the agent loop
