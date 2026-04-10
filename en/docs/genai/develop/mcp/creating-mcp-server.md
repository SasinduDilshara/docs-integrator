---
sidebar_position: 1
title: Creating an MCP Server
description: Create Model Context Protocol servers with the ballerina/mcp module to expose your integrations as tools for AI assistants.
---

# Creating an MCP Server

The Model Context Protocol (MCP) is an open standard that lets AI assistants discover and call your integration functions through a unified interface. By creating an MCP server in WSO2 Integrator, you expose your APIs, databases, and services as tools that any MCP-compatible client -- Claude Desktop, GitHub Copilot, or custom agents -- can use.

In WSO2 Integrator, MCP servers are built with the `ballerina/mcp` module, which ships with the distribution. You do not need any external dependency to create a fully functional MCP server.

## Simple MCP Server

The simplest MCP server is a `mcp:Service` attached to a `mcp:Listener`. Ballerina automatically derives each tool's name, description, and input schema from the `remote function` signature and its doc comments.

```ballerina
import ballerina/log;
import ballerina/mcp;

type Weather record {|
    string condition;
    decimal temperatureC;
|};

listener mcp:Listener mcpListener = new (9090);

service mcp:Service /mcp on mcpListener {

    # Get current weather for a city.
    #
    # + city - City name (e.g., "New York", "Tokyo")
    # + return - Current weather data for the specified city
    remote function getCurrentWeather(string city) returns Weather|error {
        log:printInfo(string `Weather requested for ${city}`);
        return {condition: "Sunny", temperatureC: 22.5};
    }

    # Get weather forecast for upcoming days.
    #
    # + location - City name
    # + days - Number of days to include in the forecast
    # + return - Forecast records, one per day
    remote function getWeatherForecast(string location, int days)
            returns Weather[]|error {
        return [{condition: "Cloudy", temperatureC: 18.0}];
    }
}
```

What happens under the hood:

- Each `remote function` becomes an MCP tool.
- The function name becomes the tool name (`getCurrentWeather`, `getWeatherForecast`).
- The doc comment becomes the tool description.
- The `+ paramName - description` lines become parameter descriptions.
- The parameter types become the JSON input schema.
- The return type becomes the tool's output schema.

Run the service with `bal run`, and it will accept MCP requests at `http://localhost:9090/mcp`.

## Writing Good Tool Definitions

MCP clients rely on your doc comments to decide when and how to call a tool. Write descriptions that are specific about inputs, outputs, and side effects.

```ballerina
service mcp:Service /mcp on mcpListener {

    # Search the product catalog by keyword. Returns up to 10 matching products
    # with their name, price, and availability.
    #
    # + query - Search keyword or product name
    # + category - Category filter: "electronics", "clothing", "home", or "all"
    # + maxPrice - Maximum price in USD. Pass () to disable the filter.
    # + return - Matching products
    remote function searchProducts(string query, string category = "all",
            decimal? maxPrice = ()) returns Product[]|error {
        // call your catalog API
        return [];
    }

    # Create a new support ticket in the internal ticketing system.
    # This action creates a real ticket and notifies the support team.
    #
    # + customerId - Customer identifier
    # + subject - Ticket subject line
    # + description - Detailed description of the issue
    # + priority - "low", "medium", "high", or "critical"
    # + return - The created ticket record
    remote function createSupportTicket(string customerId, string subject,
            string description, string priority = "medium")
            returns Ticket|error {
        // call your ticketing system
        return {id: "TKT-001", subject, priority};
    }
}

type Product record {|
    string name;
    decimal price;
    boolean inStock;
|};

type Ticket record {|
    string id;
    string subject;
    string priority;
|};
```

## Advanced MCP Server

When you need full control over tool discovery and dispatch -- for example, when tool definitions come from configuration or a database -- use `mcp:AdvancedService`. You implement `onListTools` to describe your tools manually and `onCallTool` to dispatch invocations.

```ballerina
import ballerina/mcp;

type Weather record {|
    string condition;
    decimal temperatureC;
|};

function getCurrentWeather(string city) returns Weather|error {
    return {condition: "Sunny", temperatureC: 22.5};
}

listener mcp:Listener mcpListener = new (9090);

service mcp:AdvancedService /mcp on mcpListener {

    isolated remote function onListTools()
            returns mcp:ListToolsResult|mcp:ServerError => {
        tools: [
            {
                name: "getCurrentWeather",
                description: "Get current weather conditions for a location",
                inputSchema: {
                    "type": "object",
                    "properties": {
                        "city": {
                            "type": "string",
                            "description": "City name"
                        }
                    },
                    "required": ["city"]
                }
            }
        ]
    };

    isolated remote function onCallTool(mcp:CallToolParams params, mcp:Session? session)
            returns mcp:CallToolResult|mcp:ServerError {
        do {
            if params.name == "getCurrentWeather" {
                record {|string city;|} args = check params.arguments.cloneWithType();
                Weather weather = check getCurrentWeather(args.city);
                return {
                    content: [{'type: "text", text: weather.toJsonString()}]
                };
            }
        } on fail {
            return error("Invalid arguments");
        }
        return error("Unknown tool: " + params.name);
    }
}
```

Note that content items use the field name `'type` (with a leading quote) because `type` is a reserved keyword in Ballerina.

### When to Use Each Style

| Use `mcp:Service` when... | Use `mcp:AdvancedService` when... |
|---|---|
| Your tools map cleanly to static `remote function`s | Tools are defined dynamically at runtime |
| You want Ballerina to derive schemas automatically | You need custom input schemas or validation logic |
| You want the simplest possible MCP server | You want to route calls to a dispatch table |

## Error Handling

Return informative error messages so AI assistants can reason about failures and suggest alternatives.

```ballerina
service mcp:Service /mcp on mcpListener {

    # Retrieve an invoice by ID.
    #
    # + invoiceId - Invoice identifier (format INV-XXXXX)
    # + return - Invoice details, or an error if not found
    remote function getInvoice(string invoiceId) returns json|error {
        if !invoiceId.startsWith("INV-") {
            return error(string `Invalid invoice ID '${invoiceId}'. ` +
                    "Expected format INV-XXXXX.");
        }
        // call your billing API
        return {id: invoiceId, amount: 100.00};
    }
}
```

## Configuring for Claude Desktop

Claude Desktop can connect to your MCP server over HTTP.

```json
{
  "mcpServers": {
    "weather-service": {
      "url": "http://localhost:9090/mcp"
    }
  }
}
```

## What's Next

- [Building AI Agents with MCP Servers](/docs/genai/develop/mcp/agents-with-mcp) -- Connect agents to MCP tools
- [Exposing MCP Servers](/docs/genai/mcp/exposing-mcp-servers) -- Detailed server patterns
- [MCP Security](/docs/genai/mcp/mcp-security) -- Secure your MCP endpoints
- [MCP Overview](/docs/genai/mcp/overview) -- Protocol concepts and architecture
