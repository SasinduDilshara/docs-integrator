---
sidebar_position: 1
title: MCP Integration
description: Reference for MCP features in WSO2 Integrator — exposing services as MCP servers and consuming MCP servers from AI agents.
---

# MCP Integration

WSO2 Integrator can be both an **MCP server** (exposing your integrations as standardised tools for AI assistants) and an **MCP client** (using tools published by other MCP servers from inside your own AI agents).

This section is the feature reference. For the protocol concepts see [What is MCP?](/docs/genai/key-concepts/what-is-mcp).

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [Exposing a Service as MCP](exposing-as-mcp.md) | Build an MCP service in BI — pick a listener, add tools, configure each tool's name, description, and parameters. | **Artifacts** → **MCP Service**. |
| [Consuming MCP from an Agent](consuming-mcp-from-agent.md) | Add an MCP server as a tool source for an AI Agent, optionally filtered to specific tools. | Agent canvas → **+ Add Tool** → **Use MCP Server**. |

The two sides of MCP map cleanly to the two pages — the wizard builds the server, the agent's Add Tool dialog consumes it.

## When to Use MCP

| Use MCP when… | Look elsewhere when… |
|---|---|
| You want your integration's capabilities reusable by Claude Desktop, Copilot, or any other MCP client. | Only your own agents will consume the tools — local function tools work fine. |
| You want to consume tools published by community or vendor MCP servers. | The tools live behind a normal HTTP API and you can use the connector — just wrap the connector. |
| You need a standard, governed boundary between the AI surface and your tools (auth, rate limits, audit). | Tooling is internal-only and ephemeral. |

## The Two Sides Side-by-Side

```
┌────────────────────────────┐                ┌────────────────────────────┐
│  AI Assistant / Agent       │                │   AI Assistant / Agent      │
│  (Claude Desktop, Copilot,  │                │   (your AI agent)           │
│   your own AI agent)        │                │                             │
└──────────────┬──────────────┘                └──────────────┬──────────────┘
               │   MCP                                        │   MCP
               ▼                                              ▼
┌────────────────────────────┐                ┌────────────────────────────┐
│   Your WSO2 Integrator       │                │   Someone else's MCP server │
│   MCP Service                │                │   (Slack, GitHub, vendor)   │
│                              │                │                             │
│   tools = remote functions   │                │   tools = whatever they      │
│           in BI              │                │           expose             │
└────────────────────────────┘                └────────────────────────────┘
   (Exposing as MCP)                              (Consuming MCP from an agent)
```

You can do both in the same project. A single integration might expose a few tools (your CRM lookups) over MCP for outside assistants while also consuming Slack and GitHub MCP servers from its own AI agents.

## What's Next

- **[Exposing a Service as MCP](exposing-as-mcp.md)** — turn your integration into an MCP server.
- **[Consuming MCP from an Agent](consuming-mcp-from-agent.md)** — let your agent use tools from any MCP server.
- **[What is MCP?](/docs/genai/key-concepts/what-is-mcp)** — protocol concepts.
