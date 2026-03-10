---
sidebar_position: 1
title: "MCP Overview"
description: "Understand the Model Context Protocol and how it connects LLMs to your integrations."
---

# MCP Overview

The Model Context Protocol (MCP) is an open standard that gives LLMs a standardized way to discover, understand, and invoke external tools. With WSO2 Integrator, you can both expose your integrations as MCP servers and connect your agents to external MCP servers — turning your enterprise services into tools that any AI application can use.

## What Problem Does MCP Solve?

Without MCP, every LLM application needs custom glue code to connect to each external tool. This creates tight coupling, duplicated integration logic, and fragile connections that break when APIs change.

MCP solves this by defining a common protocol:

```
┌─────────────────┐         ┌─────────────────┐
│   MCP Client    │         │   MCP Server     │
│ (LLM / Agent)   │◄──MCP──►│ (Your Service)   │
│                 │         │                  │
│ - Discovers     │         │ - Exposes tools  │
│   available     │         │ - Defines schemas│
│   tools         │         │ - Executes logic │
│ - Invokes tools │         │ - Returns results│
└─────────────────┘         └─────────────────┘
```

Any MCP client can connect to any MCP server. Your integration, once exposed as an MCP server, becomes usable by Claude, ChatGPT, custom agents, or any other MCP-compatible application.

## Core Concepts

### Tools

Tools are the functions your MCP server exposes. Each tool has a name, description, parameter schema, and return type. The LLM reads the description to decide when and how to call the tool.

```
Tool: getCurrentWeather
├── Description: "Get the current weather for a given location"
├── Parameters:
│   └── location (string, required): "City name or coordinates"
└── Returns: WeatherData
```

### Resources

Resources are read-only data sources your MCP server can provide. Unlike tools (which perform actions), resources supply context — configuration files, database schemas, documentation, or any reference data the LLM might need.

### Prompts

MCP servers can expose prompt templates — pre-built instructions that guide the LLM for specific tasks. Clients can discover and use these prompts to ensure consistent behavior.

## Two Roles in WSO2 Integrator

WSO2 Integrator supports both sides of the MCP protocol:

| Role | What it does | Use case |
|------|-------------|----------|
| **MCP Server** | Exposes your integrations as tools that LLMs can call | Make your enterprise services (databases, APIs, workflows) available to any AI application |
| **MCP Client** | Connects your agents to external MCP servers | Let your agents use tools from other MCP-compatible services |

### As an MCP Server

You take an existing service or automation built in WSO2 Integrator and expose it as an MCP server. The integration handles tool discovery, schema generation, and execution.

```ballerina
import ballerinax/ai.mcp;

@mcp:ServiceConfig {
    name: "weather-service",
    version: "1.0.0",
    description: "Weather data tools for LLM consumption"
}
service on new mcp:Listener(3000) {
    // Tools defined here become discoverable by any MCP client
}
```

### As an MCP Client

Your agent connects to external MCP servers to discover and invoke their tools. This lets your integration leverage capabilities from any MCP-compatible service without writing custom connector code.

```ballerina
import ballerinax/ai.mcp;

mcp:Client weatherMcp = check new ({
    url: "http://localhost:3000/mcp",
    transport: "sse"
});

// Discover available tools
mcp:Tool[] tools = check weatherMcp->listTools();
```

## Transport Options

MCP supports two transport mechanisms:

| Transport | Protocol | Best for |
|-----------|----------|----------|
| **SSE** (Server-Sent Events) | HTTP | Remote servers, cloud deployments, web-based clients |
| **stdio** | Standard I/O | Local tools, CLI integrations, development/testing |

:::info
For most WSO2 Integrator deployments, use SSE transport. It works over HTTP, supports authentication, and is compatible with cloud infrastructure. Use stdio only for local development tools or CLI-based agents.
:::

## How MCP Fits Into Your Architecture

MCP complements the other GenAI capabilities in WSO2 Integrator:

- **Agents** call MCP tools as part of their reasoning loop. See [Tool Binding](/docs/genai/agents/tool-binding).
- **RAG services** can be exposed as MCP tools, letting any LLM query your knowledge base. See [Build a RAG Query Service](../rag/building-rag-service.md).
- **Existing connectors** (databases, SaaS, messaging) can be wrapped as MCP servers, giving LLMs access to your enterprise data.

:::tip
Think of MCP as the "USB port" for AI. Just as USB standardized hardware connections, MCP standardizes tool connections for LLMs. Any service you expose via MCP becomes instantly usable by any compatible AI application.
:::

## What's Next

- [Expose Integrations as MCP Servers](exposing-mcp-servers.md) — Turn your services into MCP-compatible tool providers
- [Consume MCP Tools](consuming-mcp-tools.md) — Connect your agents to external MCP servers
- [Secure MCP Endpoints](security-authentication.md) — Add authentication and rate limiting to MCP services
