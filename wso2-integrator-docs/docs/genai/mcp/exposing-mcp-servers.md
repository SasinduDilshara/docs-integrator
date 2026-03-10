---
sidebar_position: 2
title: "Expose Integrations as MCP Servers"
description: "Turn your existing services and automations into MCP-compatible servers that LLMs can discover and use."
---

# Expose Integrations as MCP Servers

You can take any service or automation built in WSO2 Integrator and expose it as an MCP server. Once exposed, any MCP-compatible client — Claude, ChatGPT, custom agents, or other AI applications — can discover your tools and invoke them. This guide walks you through the full process.

## Step 1: Create the Project

1. Open VS Code and launch the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `mcp_weather_service`.
4. Choose **Service** as the project type.

## Step 2: Create the MCP Service

Define an MCP service with the `@mcp:ServiceConfig` annotation. This provides metadata that MCP clients use during discovery.

```ballerina
import ballerinax/ai.mcp;
import ballerina/http;
import ballerina/log;

@mcp:ServiceConfig {
    name: "weather-service",
    version: "1.0.0",
    description: "Provides current weather data and forecasts for cities worldwide"
}
service on new mcp:Listener(3000) {

}
```

## Step 3: Define Tools

Each tool is a resource function annotated with `@mcp:Tool`. The annotation includes the tool name, description, and parameter definitions that the LLM reads to understand how to call the tool.

```ballerina
@mcp:ServiceConfig {
    name: "weather-service",
    version: "1.0.0",
    description: "Provides current weather data and forecasts for cities worldwide"
}
service on new mcp:Listener(3000) {

    @mcp:Tool {
        name: "getCurrentWeather",
        description: "Get the current weather conditions for a specific city. Returns temperature, humidity, and conditions.",
        parameters: {
            location: {
                type: "string",
                description: "City name (e.g., 'London', 'New York', 'Tokyo')",
                required: true
            },
            units: {
                type: "string",
                description: "Temperature units: 'celsius' or 'fahrenheit'. Defaults to 'celsius'.",
                required: false
            }
        }
    }
    resource function post getCurrentWeather(string location, string units = "celsius")
            returns WeatherData|error {
        return getWeather(location, units);
    }

    @mcp:Tool {
        name: "getForecast",
        description: "Get a 5-day weather forecast for a specific city.",
        parameters: {
            location: {
                type: "string",
                description: "City name",
                required: true
            },
            days: {
                type: "integer",
                description: "Number of forecast days (1-5). Defaults to 3.",
                required: false
            }
        }
    }
    resource function post getForecast(string location, int days = 3)
            returns ForecastData|error {
        return getForecast(location, days);
    }
}
```

:::tip
Write tool descriptions as if you are explaining the function to a colleague who has never seen your code. The LLM uses these descriptions to decide when to call each tool, so clarity matters more than brevity.
:::

## Step 4: Implement Tool Logic

Behind each tool, implement the actual business logic. This is standard Ballerina code — you can call connectors, query databases, or invoke other services.

```ballerina
type WeatherData record {|
    string location;
    float temperature;
    float humidity;
    string conditions;
    string units;
|};

type ForecastData record {|
    string location;
    DayForecast[] forecast;
|};

type DayForecast record {|
    string date;
    float high;
    float low;
    string conditions;
|};

configurable string weatherApiKey = ?;

final http:Client weatherApi = check new ("https://api.weatherservice.com/v1");

function getWeather(string location, string units) returns WeatherData|error {
    json response = check weatherApi->get(
        string `/current?q=${location}&units=${units}&key=${weatherApiKey}`
    );
    // Parse and return weather data
    return {
        location: location,
        temperature: check response.temp,
        humidity: check response.humidity,
        conditions: check response.description,
        units: units
    };
}
```

## Step 5: Configure Transport

MCP supports SSE (HTTP-based) and stdio transports. For most deployments, use SSE:

### SSE Transport (Recommended)

```ballerina
// SSE is the default transport — the listener handles it automatically
service on new mcp:Listener(3000) {
    // MCP endpoint available at http://localhost:3000/mcp
}
```

### stdio Transport

Use stdio for local development tools or CLI-based integrations:

```ballerina
service on new mcp:Listener({transport: "stdio"}) {
    // Communicates via standard input/output
}
```

:::info
SSE transport is recommended for production. It supports authentication, works behind load balancers, and is compatible with cloud deployments. See [Secure MCP Endpoints](security-authentication.md) for adding authentication.
:::

## Step 6: Test with MCP Inspector

MCP Inspector is a debugging tool that lets you interact with your MCP server directly:

1. Start your MCP server:
   ```bash
   bal run
   ```

2. Open MCP Inspector and connect to `http://localhost:3000/mcp`.

3. You should see your tools listed with their descriptions and parameter schemas.

4. Test each tool by providing sample parameters and verifying the response.

```bash
# Alternatively, test with curl
curl -X POST http://localhost:3000/mcp/tools/getCurrentWeather \
  -H "Content-Type: application/json" \
  -d '{"location": "London", "units": "celsius"}'
```

## Exposing Existing Services

If you already have an HTTP service, you can expose it as an MCP server alongside the existing endpoints. Add the MCP service as a separate listener on a different port:

```ballerina
// Existing HTTP service
service /api on new http:Listener(8080) {
    resource function get weather(string city) returns WeatherData|error {
        return getWeather(city, "celsius");
    }
}

// MCP server exposing the same logic
@mcp:ServiceConfig {
    name: "weather-tools",
    version: "1.0.0",
    description: "Weather tools for LLM consumption"
}
service on new mcp:Listener(3000) {
    @mcp:Tool {
        name: "getCurrentWeather",
        description: "Get the current weather for a city",
        parameters: {
            location: {type: "string", description: "City name", required: true}
        }
    }
    resource function post getCurrentWeather(string location) returns WeatherData|error {
        return getWeather(location, "celsius");
    }
}
```

:::warning
When exposing existing services via MCP, review which operations are safe for LLM invocation. Avoid exposing destructive operations (delete, update) without proper guardrails and confirmation flows.
:::

For production deployment of your MCP server, see [Deploy & Operate](/docs/deploy-operate).

## What's Next

- [Consume MCP Tools](consuming-mcp-tools.md) — Connect your agents to MCP servers
- [Secure MCP Endpoints](security-authentication.md) — Add authentication and rate limiting
- [MCP Overview](overview.md) — Review MCP concepts and architecture
