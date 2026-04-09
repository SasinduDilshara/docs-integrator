---
title: "Exposing Services as MCP Server"
description: "Turn your existing services and automations into MCP-compatible servers that LLMs can discover and use."
---

# Exposing Services as MCP Server

You can take any service or automation built in WSO2 Integrator and expose it as an MCP server. Once exposed, any MCP-compatible client — Claude, ChatGPT, custom agents, or other AI applications — can discover your tools and invoke them.

:::note What is an MCP Service?
An MCP service acts as a bridge between AI assistants and external systems. It exposes a set of tools — functions that AI assistants can call to perform specific tasks like fetching data, executing operations, or interacting with APIs. The AI assistant discovers available tools through the MCP protocol and invokes them as needed during conversations.
:::

## Step 1: Create the Project

1. Click on the **BI** icon in the sidebar.
2. Click on the **Create New Integration** button.
3. Enter the project name as `mcp_weather_service`.
4. Select a directory location by clicking on the **Select Path** button.
5. Click **Create New Integration** to generate the project.

![Create a new integration project](/img/genai/mcp/create-a-new-integration-project.gif)

## Step 2: Create the MCP Service Artifact

1. In the design screen, click on **+ Add Artifact**.
2. Select **MCP Service** under the **AI Integration** artifact category.
3. Enter the service name as `Weather MCP`.
4. Leave the other fields at their default values.
5. Click **Create** to open the MCP service editor.

![Create an MCP service](/img/genai/mcp/create-an-mcp-service.gif)

### @mcp:ServiceConfig Annotation

In pro-code, the MCP service is represented with the `@mcp:ServiceConfig` annotation, which provides metadata that MCP clients use during discovery:

```ballerina
import ballerina/mcp;
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

## Step 3: Define Tools with Remote Functions

Tools are the core building blocks of an MCP service. Each tool represents a specific capability that AI assistants can invoke.

### Using the Visual Designer

1. Under the **Tools** section of the MCP service, click on **+ Add Tool**.
2. Configure the tool with the following details:
   - **Tool Name**: `getCurrentWeather`
   - **Description**: `Retrieve current weather data for a specified location.`
3. Under **Input Parameters**, add a parameter:
   - **Name**: `location`
   - **Type**: `string`
   - **Description**: `The location for which to retrieve the current weather.`
4. Set the **Return Type** to `string`.
5. Click **Save** to add the tool.

![Define tools for the MCP service](/img/genai/mcp/define-a-tool.gif)

:::tip
Tool descriptions are critical — they help AI assistants understand when and how to use each tool. Write clear, concise descriptions that explain what the tool does and what input it expects.
:::

### Pro-Code Tool Definition

Each tool is a resource function annotated with `@mcp:Tool`. The annotation includes the tool name, description, and parameter definitions that the LLM reads to understand how to call the tool:

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

### Parameter Schemas from Doc Comments

The official Ballerina MCP approach uses doc comments on remote functions to generate parameter schemas automatically. The following example shows how doc comments serve as the tool description and parameter documentation:

```ballerina
import ballerina/mcp;
import ballerina/time;

type Weather record {|
    string location;
    decimal temperature;
    int humidity;
    int pressure;
    string condition;
    string timestamp;
|};

type ForecastItem record {|
    string date;
    int high;
    int low;
    string condition;
    int precipitationChance;
    int windSpeed;
|};

type WeatherForecast record {|
    string location;
    ForecastItem[] forecast;
|};

listener mcp:Listener mcpListener = new (9090);

service mcp:Service /mcp on mcpListener {
    # Get current weather for a city.
    # + location - The name of the city
    # + return - Current weather information
    remote function getCurrentWeather(string location) returns Weather {
        return {
            location: location,
            temperature: 22.5d,
            humidity: 65,
            pressure: 1013,
            condition: "Partly cloudy",
            timestamp: time:utcNow().toString()
        };
    }

    # Get weather forecast for a city.
    # + location - The name of the city
    # + days - Number of forecast days (1-7)
    # + return - Weather forecast
    remote function getWeatherForecast(string location, int days) returns WeatherForecast {
        // implementation
    }
}
```

In this approach, the `#` doc comments on each remote function automatically become the tool description that MCP clients see during discovery, and the `+ paramName` annotations define parameter descriptions.

## Step 4: Implement Tool Logic

Implement the logic that executes when the tool is invoked. In the visual designer:

1. Click on the **getCurrentWeather** tool to open its implementation flow diagram.
2. Hover over the flow line and click the **+** icon to open the side panel.
3. Select **If** from the **Control** section.
4. In the condition field, enter: `location == "New York"`
5. Click **Add Else Block** to handle other locations.
6. Click **Save** to add the conditional block.
7. In the **If** block, add a **Return** node with value `"Sunny, 72F"`.
8. In the **Else** block, add a **Return** node with value `"Weather data not available for this location."`.

![Implement tool logic](/img/genai/mcp/implement-tool-logic.gif)

:::note
This example uses simple conditional logic for demonstration purposes. In a real-world scenario, you would integrate with actual weather APIs using [HTTP connectors](/docs/connectors) or other data sources available in WSO2 Integrator.
:::

### Pro-Code Implementation

Behind each tool, implement the actual business logic. This is standard Ballerina code — you can call connectors, query databases, or invoke other services:

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

### Adding More Tools

You can extend your MCP service with additional tools to provide more capabilities:

1. Click on `mcp:Service` in the left panel to return to the MCP service overview.
2. Click on **+ Tool** and repeat the process from Step 3 to define and implement more tools.

![Add more tools](/img/genai/mcp/add-more-tools.gif)

## Step 5: Transport Configuration

MCP supports SSE (HTTP-based) and stdio transports.

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
SSE transport is recommended for production. It supports authentication, works behind load balancers, and is compatible with cloud deployments.
:::

## Step 6: Exposing Existing Services

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

## Step 7: Testing with MCP Inspector

Once you have defined and implemented your tools, test the MCP service.

### Using the Built-in Try It Feature

1. In the left panel, click on `mcp:Service` to return to the MCP service overview.
2. Click on the **Try It** button in the top-right corner.
3. When prompted, install the **MCP Inspector** extension by clicking **Install**.
4. Once the installation is complete, click **Try It** again to open the MCP Inspector.
5. In the MCP Inspector:
   - Click **Connect** to establish a connection with your MCP service.
   - Click **List Tools** to view all tools exposed by the service.

:::info Connecting AI Assistants
Once your MCP service is running, AI assistants can connect to it using the MCP protocol. The service URL and available tools will be discoverable through the standard MCP handshake process.
:::

### Testing with curl

You can also test directly with curl:

```bash
# Start the MCP server
bal run

# Test a tool
curl -X POST http://localhost:3000/mcp/tools/getCurrentWeather \
  -H "Content-Type: application/json" \
  -d '{"location": "London", "units": "celsius"}'
```

## What's Next

- [Building AI Agents with MCP Servers](building-agents-with-mcp-servers.md) -- Bind MCP tools to agents and invoke tools directly
