---
sidebar_position: 2
title: Building AI Agents with MCP Servers
description: Connect WSO2 Integrator AI agents to MCP servers using the ai:McpToolKit for standardized tool discovery and invocation.
---

# Building AI Agents with MCP Servers

MCP (Model Context Protocol) servers expose tools through a standardized interface that any compatible client can discover and call. By connecting your AI agents to MCP servers, you can extend agent capabilities without writing custom connector code for each service.

In WSO2 Integrator, agents consume MCP servers through the `ai:McpToolKit`. A single toolkit instance exposes every tool from an MCP endpoint to the agent, and you can combine multiple toolkits (or mix them with local tools) in the same agent.

## Connecting an Agent to an MCP Server

The core building block is `ai:McpToolKit`. Pass the MCP server's URL to its constructor, then include the toolkit in your agent's `tools` array.

```ballerina
import ballerina/ai;
import ballerina/io;

// Connect to a running MCP server
final ai:McpToolKit weatherMcpConn = check new ("http://localhost:9090/mcp");

final ai:Agent weatherAgent = check new (
    systemPrompt = {
        role: "Weather-aware AI Assistant",
        instructions: string `You are a smart AI assistant that can assist 
            a user based on accurate and timely weather information.`
    },
    tools = [weatherMcpConn],
    model = check ai:getDefaultModelProvider()
);

public function main() returns error? {
    while true {
        string userInput = io:readln("User (or 'exit' to quit): ");
        if userInput == "exit" {
            break;
        }
        string response = check weatherAgent.run(userInput);
        io:println("Agent: ", response);
    }
}
```

When the LLM reasons about the user's request, it sees every tool the MCP server advertises and can invoke any of them through the agent's execution loop. You never need to write wiring code for individual tools.

## Filtering to Specific Tools

When an MCP server exposes many tools, you can narrow the toolkit to just the tools you care about by passing a list of tool names as the second argument. This keeps the LLM's decision space small and leads to cleaner tool selection.

```ballerina
final ai:McpToolKit readOnlyWeatherConn = check new (
    "http://localhost:9090/mcp",
    ["getCurrentWeather", "getWeatherForecast"]
);
```

Only the listed tools are exposed to the agent; everything else the server advertises is hidden.

## Combining Multiple MCP Servers

Agents can use any number of MCP toolkits. Create one toolkit per server and include them all in the `tools` array.

```ballerina
import ballerina/ai;

final ai:McpToolKit weatherMcpConn = check new ("http://localhost:9090/mcp");
final ai:McpToolKit calendarMcpConn = check new ("http://localhost:9091/mcp");
final ai:McpToolKit emailMcpConn = check new ("http://localhost:9092/mcp");

final ai:Agent assistantAgent = check new (
    systemPrompt = {
        role: "Personal Assistant",
        instructions: "You help users plan their day using weather, " +
                "calendar, and email information."
    },
    tools = [weatherMcpConn, calendarMcpConn, emailMcpConn],
    model = check ai:getDefaultModelProvider()
);
```

## Mixing MCP Toolkits with Local Tools

You can freely mix `ai:McpToolKit` instances with locally defined `@ai:AgentTool` functions in the same `tools` array. This is useful when some capabilities live in external MCP servers while others are implemented directly in your Ballerina codebase.

```ballerina
import ballerina/ai;

configurable string crmBaseUrl = ?;

# Look up customer details from the internal CRM by customer ID.
#
# + customerId - Customer identifier
# + return - The customer record, or an error if the lookup fails
@ai:AgentTool
isolated function getCustomerDetails(string customerId) returns json|error {
    // call your CRM
    return {id: customerId, name: "Jane Doe"};
}

# Create a new support ticket in the internal ticketing system.
#
# + subject - Ticket subject line
# + description - Detailed description of the issue
# + return - The created ticket record
@ai:AgentTool
isolated function createSupportTicket(string subject, string description)
        returns json|error {
    return {id: "TKT-001", subject};
}

final ai:McpToolKit slackMcpConn = check new ("http://localhost:9093/mcp");

final ai:Agent supportAgent = check new (
    systemPrompt = {
        role: "Customer Support Assistant",
        instructions: "You help support agents by looking up customers, " +
                "creating tickets, and sending Slack notifications."
    },
    tools = [getCustomerDetails, createSupportTicket, slackMcpConn],
    model = check ai:getDefaultModelProvider()
);
```

Local tools and MCP tools are completely interchangeable from the agent's perspective. The LLM sees a single unified tool list.

## Connecting to Your Own MCP Servers

The `ai:McpToolKit` does not care whether an MCP server is third-party or one you built yourself. Point it at any MCP endpoint -- including a server you created with `ballerina/mcp` as shown in [Creating an MCP Server](/docs/genai/develop/mcp/creating-mcp-server).

```ballerina
import ballerina/ai;

final ai:McpToolKit inventoryMcpConn = check new ("http://localhost:8091/mcp");
final ai:McpToolKit analyticsMcpConn = check new ("http://localhost:8092/mcp");

final ai:Agent internalAgent = check new (
    systemPrompt = {
        role: "Operations Assistant",
        instructions: "You help operators inspect inventory and analytics data."
    },
    tools = [inventoryMcpConn, analyticsMcpConn],
    model = check ai:getDefaultModelProvider()
);
```

## Exposing the Agent as an HTTP Service

Wrap your MCP-powered agent in an HTTP service to make it available to other applications.

```ballerina
import ballerina/ai;
import ballerina/http;

final ai:McpToolKit weatherMcpConn = check new ("http://localhost:9090/mcp");

final ai:Agent weatherAgent = check new (
    systemPrompt = {
        role: "Weather-aware AI Assistant",
        instructions: "Use the weather tools to answer user questions."
    },
    tools = [weatherMcpConn],
    model = check ai:getDefaultModelProvider()
);

type ChatRequest record {|
    string message;
|};

type ChatResponse record {|
    string message;
|};

service /api on new http:Listener(8080) {

    resource function post chat(@http:Payload ChatRequest request)
            returns ChatResponse|error {
        string response = check weatherAgent.run(request.message);
        return {message: response};
    }
}
```

Note that you call the agent with `agent.run(...)` -- a regular method call. Behind the scenes the agent runtime handles the MCP tool dispatch, the LLM reasoning loop, and the response synthesis.

## Tips for Production

- **Keep tool lists small.** Prefer a filtered `ai:McpToolKit` over exposing every tool the server advertises. LLMs pick better tools when given fewer options.
- **Run MCP servers close to the agent.** MCP calls happen in the agent's hot path, so network latency between the agent and its MCP servers directly affects user response time.
- **Use `configurable` for endpoints.** Hard-coding MCP URLs makes environments harder to manage. Pull them from `Config.toml` instead.

```toml
# Config.toml
weatherMcpUrl = "http://localhost:9090/mcp"
```

```ballerina
configurable string weatherMcpUrl = ?;

final ai:McpToolKit weatherMcpConn = check new (weatherMcpUrl);
```

## What's Next

- [Creating an MCP Server](/docs/genai/develop/mcp/creating-mcp-server) -- Build your own MCP server
- [Consuming MCP Tools](/docs/genai/mcp/consuming-mcp-tools) -- Detailed MCP client patterns
- [MCP Security](/docs/genai/mcp/mcp-security) -- Secure MCP connections
- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build agents from scratch
