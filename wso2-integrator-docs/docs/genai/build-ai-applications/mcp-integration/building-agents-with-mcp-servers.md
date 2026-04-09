---
title: "Building Agents with MCP Servers"
description: "Connect AI agents to MCP servers to give them access to external tools and services."
---

# Building Agents with MCP Servers

The most powerful use of MCP is binding MCP tools to an agent. The agent discovers the tools, decides when to call them during its reasoning loop, and incorporates the results into its responses. This guide shows how to connect agents to one or more MCP servers using `ai:McpToolKit`.

## ai:McpToolKit for Connecting Agents to MCP Servers

The `ai:McpToolKit` type provides a streamlined way to connect an agent to an MCP server. You pass the MCP server URL and then use the toolkit directly in the agent's `tools` configuration:

```ballerina
import ballerina/ai;
import ballerina/io;

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
        if userInput == "exit" { break; }
        string response = check weatherAgent->run(userInput);
        io:println("Agent: " + response);
    }
}
```

When a user asks "What's the weather like in Tokyo?", the agent:

1. Reads the available tools and sees `getCurrentWeather`
2. Decides to call it with `{"location": "Tokyo"}`
3. Receives the weather data from the MCP server
4. Formulates a natural language response using the data

## Weather Assistant Example

This walkthrough creates a Weather AI Assistant that connects to an MCP server for real-time weather data.

:::note Prerequisites
Before you begin, ensure you have a running MCP Server connected to a weather service. You can set up one using [this guide](https://github.com/xlight05/mcp-openweathermap).
:::

### Visual Designer Walkthrough

**Create the AI agent** by following Steps 1 to 5 in the [Create a Chat Agent](../ai-agents/creating-an-ai-agent.md) guide. Configure the agent with:

**Role:**

```text
Weather AI Assistant
```

**Instructions:**

```text
You are Nova, a smart AI assistant dedicated to providing accurate and timely weather information.

Your primary responsibilities include:
- Current Weather: Provide detailed and user-friendly current weather information for a given location.
- Weather Forecast: Share reliable weather forecasts according to user preferences (e.g., hourly, daily).

Guidelines:
- Always communicate in a natural, friendly, and professional tone.
- Provide concise summaries unless the user explicitly requests detailed information.
- Confirm location details if ambiguous and suggest alternatives when data is unavailable.
```

**Step 1: Add the MCP server**

1. In Agent Flow View, click the **+** button at the bottom-right of the `AI Agent` box.
2. Under **Add Tools** section, select **Use MCP Server**.
3. Provide the necessary configuration details, then click **Save Tool**.

![Add MCP Server](/img/genai/agents/external-endpoints/ai-agent-add-mcp-server.gif)

**Step 2: Customize the MCP server**

You can further customize the MCP configuration to include additional weather tools to suit your use case.

![Edit MCP Server](/img/genai/agents/external-endpoints/ai-agent-edit-mcp-server.gif)

**Step 3: Interact with the agent**

1. Click the **Chat** button located at the top-left corner of the interface.
2. You will be prompted to run the integration. Click **Run Integration**.
3. Start chatting with your weather assistant.

![Interact with the weather agent](/img/genai/agents/external-endpoints/ai-agent-interact-mcp-server.gif)

### Pro-Code Example

This example builds the same weather agent in pro-code:

```ballerina
import ballerina/ai;
import ballerina/mcp;
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

## Multiple MCP Server Consumption

Your agent can consume tools from multiple MCP servers simultaneously. This allows a single agent to have access to a wide range of capabilities from different services:

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

## Routing Between MCP Servers

When an agent has access to multiple MCP servers, the `ai:MultiMcpExecutor` handles routing tool calls to the correct server automatically. The executor maintains a mapping between tool names and their source MCP server, so when the agent decides to call a tool, the executor forwards the request to the right server.

This means you can combine tools from different domains (weather, calendar, email, databases) into a single agent without worrying about which server handles which tool. The routing is transparent to both the agent and the LLM.

:::warning
If an MCP server is down, your agent should still respond — just without the data from that tool. Design your system prompt to instruct the agent on how to handle tool failures gracefully.
:::

## MCP Client Authentication

### Bearer Token

For MCP servers that require authentication, provide a bearer token in the client configuration:

```ballerina
configurable string mcpBearerToken = ?;

final mcp:Client secureMcpClient = check new ({
    url: "https://tools.example.com/mcp",
    transport: "sse",
    auth: {
        token: mcpBearerToken
    }
});
```

### OAuth 2.0 Client Credentials

For production deployments with OAuth 2.0:

```ballerina
final mcp:Client secureMcpClient = check new ({
    url: "https://tools.example.com/mcp",
    transport: "sse",
    auth: {
        type: "oauth2",
        tokenUrl: "https://identity.example.com/oauth2/token",
        clientId: clientId,
        clientSecret: clientSecret,
        scopes: ["mcp:read", "mcp:execute"]
    }
});
```

## Direct Tool Invocation

In some cases, you may want to call MCP tools directly without going through an agent's reasoning loop. Use `callTool()` for programmatic access:

```ballerina
function getWeather(string city) returns json|error {
    json result = check mcpClient->callTool("getCurrentWeather", {
        "location": city,
        "units": "celsius"
    });
    return result;
}
```

You can also discover available tools at runtime using `listTools()`:

```ballerina
function discoverTools() returns error? {
    mcp:Tool[] tools = check mcpClient->listTools();

    foreach mcp:Tool tool in tools {
        log:printInfo("Available tool",
            name = tool.name,
            description = tool.description
        );
    }
}
```

:::tip
Always wrap MCP tool calls in error handling. External MCP servers may be unavailable, rate-limited, or return unexpected results. Treat them like any external service call.
:::

## Error Handling

MCP tool calls can fail for various reasons. Handle errors gracefully so your integration can recover:

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
        // Return a structured error the caller can interpret
        return {"error": true, "message": string `Tool '${toolName}' is currently unavailable.`};
    }
}
```

## Testing

Test your agent with MCP integration by running the MCP server and agent service:

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

- [Creating an MCP Server](exposing-services-as-mcp-server.md) -- Build your own MCP servers
- [Creating an AI Agent](../ai-agents/creating-an-ai-agent.md) -- Learn the fundamentals of agent creation
