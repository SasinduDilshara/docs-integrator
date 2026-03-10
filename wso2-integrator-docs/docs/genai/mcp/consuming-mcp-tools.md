---
sidebar_position: 3
title: "Consume MCP Tools"
description: "Connect your agents and integrations to external MCP servers to discover and invoke tools."
---

# Consume MCP Tools

When you connect your agent or integration to an external MCP server, your code gains access to every tool that server exposes — without writing custom connector logic. You discover tools at runtime, invoke them with structured parameters, and handle the results like any other function call.

This guide shows you how to connect to MCP servers, discover their tools, and invoke them from your integrations.

## Connect to an MCP Server

Create an MCP client that points to the target server:

```ballerina
import ballerinax/ai.mcp;
import ballerina/log;

configurable string mcpServerUrl = "http://localhost:3000/mcp";

final mcp:Client mcpClient = check new ({
    url: mcpServerUrl,
    transport: "sse"
});
```

For MCP servers that require authentication:

```ballerina
configurable string mcpApiKey = ?;

final mcp:Client mcpClient = check new ({
    url: mcpServerUrl,
    transport: "sse",
    auth: {
        token: mcpApiKey
    }
});
```

## Discover Available Tools

Once connected, list the tools the server exposes. This is useful for dynamic integrations where you do not know the available tools ahead of time:

```ballerina
function discoverTools() returns error? {
    mcp:Tool[] tools = check mcpClient->listTools();

    foreach mcp:Tool tool in tools {
        log:printInfo("Available tool",
            name = tool.name,
            description = tool.description
        );
        // Inspect parameter schema
        foreach [string, mcp:Parameter] [paramName, param] in tool.parameters.entries() {
            log:printInfo("  Parameter",
                name = paramName,
                'type = param.'type,
                required = param.required
            );
        }
    }
}
```

## Invoke Tools Directly

Call a specific tool by name with the required parameters:

```ballerina
function getWeather(string city) returns json|error {
    json result = check mcpClient->callTool("getCurrentWeather", {
        "location": city,
        "units": "celsius"
    });
    return result;
}
```

:::tip
Always wrap MCP tool calls in error handling. External MCP servers may be unavailable, rate-limited, or return unexpected results. Treat them like any external service call.
:::

## Connect an Agent to MCP Servers

The most powerful use of MCP is binding MCP tools to an agent. The agent discovers the tools, decides when to call them during its reasoning loop, and incorporates the results into its responses.

### Weather Assistant Example

This example builds an agent that uses a weather MCP server to answer weather-related questions:

```ballerina
import ballerinax/ai;
import ballerinax/ai.mcp;
import ballerina/http;

configurable string openaiApiKey = ?;
configurable string weatherMcpUrl = "http://localhost:3000/mcp";

// Connect to the weather MCP server
final mcp:Client weatherMcp = check new ({
    url: weatherMcpUrl,
    transport: "sse"
});

// Create the agent with MCP tools
final ai:Agent weatherAgent = check new ({
    model: {
        provider: "openai",
        apiKey: openaiApiKey,
        modelId: "gpt-4o"
    },
    systemPrompt: "You are a weather assistant. Use the available weather tools to answer questions about current conditions and forecasts. Always include the temperature and conditions in your response.",
    tools: check weatherMcp->listTools(),
    toolExecutor: weatherMcp
});

service /api on new http:Listener(9090) {
    resource function post chat(@http:Payload ChatRequest request) returns ChatResponse|error {
        string response = check weatherAgent->chat(request.message);
        return {reply: response};
    }
}

type ChatRequest record {|
    string message;
|};

type ChatResponse record {|
    string reply;
|};
```

When a user asks "What's the weather like in Tokyo?", the agent:

1. Reads the available tools and sees `getCurrentWeather`
2. Decides to call it with `{"location": "Tokyo"}`
3. Receives the weather data from the MCP server
4. Formulates a natural language response using the data

## Connect to Multiple MCP Servers

Your agent can consume tools from multiple MCP servers simultaneously:

```ballerina
// Connect to multiple MCP servers
final mcp:Client weatherMcp = check new ({url: "http://weather-service:3000/mcp", transport: "sse"});
final mcp:Client calendarMcp = check new ({url: "http://calendar-service:3001/mcp", transport: "sse"});
final mcp:Client emailMcp = check new ({url: "http://email-service:3002/mcp", transport: "sse"});

// Aggregate tools from all servers
function getAllTools() returns mcp:Tool[]|error {
    mcp:Tool[] allTools = [];
    allTools.push(...check weatherMcp->listTools());
    allTools.push(...check calendarMcp->listTools());
    allTools.push(...check emailMcp->listTools());
    return allTools;
}

// Create a multi-tool executor that routes to the correct server
final ai:MultiMcpExecutor executor = check new ([weatherMcp, calendarMcp, emailMcp]);

final ai:Agent assistant = check new ({
    model: {provider: "openai", apiKey: openaiApiKey, modelId: "gpt-4o"},
    systemPrompt: "You are a personal assistant with access to weather, calendar, and email tools.",
    tools: check getAllTools(),
    toolExecutor: executor
});
```

:::info
When connecting to multiple MCP servers, make sure tool names are unique across all servers. If two servers expose a tool with the same name, prefix them during registration to avoid conflicts.
:::

## Handling Tool Execution Errors

MCP tool calls can fail for various reasons. Handle errors gracefully so your agent can recover:

```ballerina
function safeToolCall(mcp:Client mcpClient, string toolName, map<json> params) returns json|error {
    do {
        json result = check mcpClient->callTool(toolName, params);
        return result;
    } on fail error e {
        log:printError("MCP tool call failed",
            tool = toolName,
            'error = e.message()
        );
        // Return a structured error the agent can interpret
        return {"error": true, "message": string `Tool '${toolName}' is currently unavailable.`};
    }
}
```

:::warning
If an MCP server is down, your agent should still respond — just without the data from that tool. Design your system prompt to instruct the agent on how to handle tool failures gracefully.
:::

## Testing MCP Tool Consumption

Test your MCP client integration by running the MCP server locally:

```bash
# Terminal 1: Start the MCP server
cd mcp_weather_service && bal run

# Terminal 2: Start your agent service
cd weather_assistant && bal run

# Terminal 3: Test the agent
curl -X POST http://localhost:9090/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the weather like in London today?"}'
```

## What's Next

- [Secure MCP Endpoints](security-authentication.md) — Add authentication and rate limiting to MCP connections
- [Expose Integrations as MCP Servers](exposing-mcp-servers.md) — Build MCP servers for your own services
- [Tool Binding (Function Calling)](/docs/genai/agents/tool-binding) — Learn more about connecting tools to agents
