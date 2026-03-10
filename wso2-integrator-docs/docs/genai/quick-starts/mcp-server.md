---
sidebar_position: 3
title: "Expose an MCP Server"
description: Create an MCP server that exposes weather tools for any LLM client to discover and call.
---

# Expose an MCP Server

**Time:** 10-15 minutes · **What you'll build:** An MCP (Model Context Protocol) server that exposes weather lookup tools, discoverable by any MCP-compatible LLM client.

You will create a service that speaks the MCP protocol, define tools with typed schemas, implement the tool logic, and test it using the MCP Inspector. Any agent or LLM client that supports MCP can then discover and call your tools automatically.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) (optional, for testing)

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `weather-mcp-server` and choose a directory.

## Step 2: Add an MCP Service Artifact

1. Click **+ Add Artifact** and select **MCP Service**.
2. Name it `WeatherMcpService`.

WSO2 Integrator generates the MCP service scaffold:

```ballerina
import ballerinax/mcp;

configurable int port = 3000;

@mcp:ServiceConfig {
    name: "weather-tools",
    version: "1.0.0"
}
service on new mcp:Listener(port) {

}
```

## Step 3: Define the Tools

Add two tools to the MCP service — one for current weather and one for a forecast:

```ballerina
import ballerinax/mcp;

configurable int port = 3000;

type WeatherResult record {|
    string city;
    float temperature;
    string condition;
    int humidity;
|};

type ForecastDay record {|
    string date;
    float high;
    float low;
    string condition;
|};

@mcp:ServiceConfig {
    name: "weather-tools",
    version: "1.0.0"
}
service on new mcp:Listener(port) {

    @mcp:Tool {
        description: "Get current weather conditions for a city"
    }
    resource function get currentWeather(string city) returns WeatherResult|error {
        // In production, call a real weather API connector
        return {
            city: city,
            temperature: 22.5,
            condition: "Partly Cloudy",
            humidity: 65
        };
    }

    @mcp:Tool {
        description: "Get a 3-day weather forecast for a city"
    }
    resource function get forecast(string city, int days = 3) returns ForecastDay[]|error {
        // In production, call a real weather API connector
        ForecastDay[] forecastDays = [];
        foreach int i in 0 ..< days {
            forecastDays.push({
                date: string `2026-03-${11 + i}`,
                high: 24.0 - <float>i,
                low: 14.0 - <float>i,
                condition: i == 0 ? "Sunny" : "Cloudy"
            });
        }
        return forecastDays;
    }
}
```

:::info
Each tool is defined as a resource function with the `@mcp:Tool` annotation. WSO2 Integrator automatically generates the JSON Schema for tool parameters and return types from your Ballerina type definitions.
:::

## Step 4: Run the MCP Server

1. Click **Run** in the VS Code toolbar.
2. The MCP server starts on the configured port (default: 3000).
3. You should see output similar to:

```
MCP server "weather-tools" v1.0.0 started on port 3000
Registered tools: currentWeather, forecast
```

## Step 5: Test with MCP Inspector

Open the MCP Inspector and connect to your server:

```bash
npx @modelcontextprotocol/inspector connect http://localhost:3000
```

The Inspector shows:
- **Server info:** weather-tools v1.0.0
- **Available tools:** `currentWeather`, `forecast`

Call a tool from the Inspector:

```json
{
  "tool": "currentWeather",
  "arguments": {
    "city": "San Francisco"
  }
}
```

Response:

```json
{
  "city": "San Francisco",
  "temperature": 22.5,
  "condition": "Partly Cloudy",
  "humidity": 65
}
```

:::tip
You can also test by connecting an agent to your MCP server. See [Consuming MCP Tools](/docs/genai/mcp/consuming-mcp-tools) to learn how agents discover and call MCP tools at runtime.
:::

## Step 6: Connect to a Real Weather API

Replace the stub data with a real connector call:

```ballerina
import ballerinax/openweathermap;

configurable string weatherApiKey = ?;

final openweathermap:Client weatherClient = check new ({apiKey: weatherApiKey});

@mcp:Tool {
    description: "Get current weather conditions for a city"
}
resource function get currentWeather(string city) returns WeatherResult|error {
    openweathermap:CurrentWeather data = check weatherClient->getCurrentWeather(city);
    return {
        city: city,
        temperature: data.main.temp,
        condition: data.weather[0].description,
        humidity: data.main.humidity
    };
}
```

## What's Next

- [MCP Overview](/docs/genai/mcp/overview) — Understand the Model Context Protocol and its role in integrations
- [Exposing Integrations as MCP Servers](/docs/genai/mcp/exposing-mcp-servers) — Advanced patterns for multi-tool MCP servers
- [Build a Conversational Agent](/docs/genai/quick-starts/conversational-agent) — Create an agent that consumes MCP tools
