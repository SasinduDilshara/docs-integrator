---
title: "How MCP Works"
description: "Understand the core concepts, client-server architecture, and transport options of the Model Context Protocol."
---

# How MCP Works

MCP follows a **client-server architecture** where AI applications act as clients that discover and invoke tools exposed by servers. This page covers the core concepts, how the protocol flows, and the transport options available.

## Core Concepts

MCP servers expose three types of capabilities:

### Tools

Tools are the primary capability in MCP. A tool is a function that the server exposes for invocation. Each tool has:

- **Name** -- A unique identifier (e.g., `getCurrentWeather`)
- **Description** -- A human-readable explanation of what the tool does, which the LLM reads to decide when to call it
- **Parameter schema** -- A JSON Schema defining the inputs the tool accepts
- **Return type** -- The structure of the tool's response

```
Tool: getCurrentWeather
  Name: "getCurrentWeather"
  Description: "Get the current weather for a given location"
  Parameters:
    location (string, required): "City name or coordinates"
  Returns: WeatherData
```

The LLM uses the tool's name and description to decide whether and when to call it. Clear, specific descriptions lead to better tool selection by the model.

### Resources

Resources are **read-only data sources** that an MCP server can provide. Unlike tools (which perform actions), resources supply context -- configuration files, database schemas, documentation, API specifications, or any reference data the LLM might need to do its job effectively.

Resources give the AI application background knowledge without requiring a tool call. For example, a server might expose a "company policies" resource that the LLM can reference when answering HR-related questions.

### Prompts

MCP servers can expose **prompt templates** -- pre-built instructions that guide the LLM for specific tasks. Clients can discover these prompts and use them to ensure consistent behavior across different AI applications that connect to the same server.

## Client-Server Architecture

MCP defines two roles:

| Role | Responsibility | Examples |
|---|---|---|
| **MCP Client** | Discovers available tools, invokes them, and processes results | AI agents, LLM applications, chatbots |
| **MCP Server** | Exposes tools, resources, and prompts for clients to consume | Your integrations, enterprise services, third-party APIs |

The interaction flow works as follows:

1. **Connection** -- The client connects to the server using one of the supported transport mechanisms.
2. **Discovery** -- The client requests a list of available tools (and optionally resources and prompts). The server responds with names, descriptions, and schemas.
3. **Invocation** -- When the LLM decides to call a tool, the client sends an invocation request with the tool name and arguments. The server executes the function and returns the result.
4. **Iteration** -- The client can make multiple tool calls during a single interaction, and the LLM can chain tool calls based on intermediate results.

```
MCP Client                          MCP Server
    |                                    |
    |--- list tools ------------------->|
    |<-- tool schemas ------------------|
    |                                    |
    |--- invoke "getCurrentWeather" ---->|
    |<-- weather data ------------------|
    |                                    |
```

## Transport Options

MCP supports two transport mechanisms for the connection between client and server:

### SSE (Server-Sent Events)

SSE runs over HTTP, making it suitable for remote servers, cloud deployments, and web-based clients. It supports standard HTTP authentication and works with existing cloud infrastructure like load balancers and API gateways.

**Best for:** Production deployments, remote services, web-accessible tools.

### stdio (Standard I/O)

stdio uses standard input and output streams for communication. The client spawns the server as a local process and communicates through stdin/stdout.

**Best for:** Local development tools, CLI-based integrations, testing environments.

For most WSO2 Integrator deployments, SSE is the recommended transport.

## MCP in Ballerina

WSO2 Integrator supports both sides of the MCP protocol. Here is an example of an MCP server that exposes a weather tool:

```ballerina
import ballerina/mcp;

listener mcp:Listener mcpListener = new (9090);

service mcp:Service /mcp on mcpListener {
    remote function getCurrentWeather(string location) returns Weather {
        return { location: location, temperature: 22.5d, humidity: 65, condition: "Partly cloudy" };
    }
}
```

In this example:

- The `mcp:Listener` creates an MCP server on port 9090.
- Each `remote function` in the service becomes a discoverable tool.
- The function signature (parameters and return type) is automatically converted into the tool's schema.
- Any MCP client can connect, discover the `getCurrentWeather` tool, and invoke it.

## How MCP Fits with Agents

MCP complements the agent architecture in WSO2 Integrator:

- **Agents** can use MCP clients to discover and call tools from external MCP servers, expanding their capabilities without custom connector code.
- **Existing integrations** can be wrapped as MCP servers, making enterprise services (databases, APIs, workflows) available to any AI application.
- **RAG services** can be exposed as MCP tools, letting any LLM query your knowledge base through the standard protocol.

## What's Next

- [Why MCP?](why-mcp.md) -- The benefits and motivation behind the MCP standard
- [What Are Tools?](../what-are-tools.md) -- How tools work within AI agents
- [What Is an AI Agent?](../what-is-an-ai-agent.md) -- The agent architecture that consumes MCP tools
