---
sidebar_position: 6
title: What is MCP?
description: A plain-English explanation of MCP — the standard plug for connecting AI assistants to your tools and data.
---

# What is MCP?

**MCP** stands for **Model Context Protocol**. It's a standard way for an AI assistant — like Claude Desktop, GitHub Copilot, or your own AI agent — to talk to external tools and data.

Think of MCP as a **universal plug**. Before MCP, every AI assistant had its own custom way of being connected to your systems. Each integration had to be built from scratch. With MCP, you publish your tools once, in one shape, and any MCP-compatible assistant can use them — no custom code on the assistant side, no custom code on your side beyond the server.

## Why Does This Matter?

Imagine you have a useful internal system — a CRM, a knowledge base, a ticketing tool. You want your AI assistants to use it: look up customers, search tickets, file new ones.

Without MCP, you would build:

- An adapter for Claude Desktop.
- A plugin for GitHub Copilot.
- A custom integration for your own AI agent.
- Another one for whatever new AI assistant launches next month.

With MCP, you build the adapter **once** as an MCP **server**, and every assistant that speaks MCP can use it. Same shape, same protocol, same tool descriptions. The plug fits everywhere.

## The Two Sides of MCP

Every MCP relationship has two sides:

| Role | What it is | Example |
|---|---|---|
| **MCP Server** | The thing that exposes tools, data, or prompts. | A service in WSO2 Integrator that lets AI assistants look up your customers. |
| **MCP Client** | The thing that uses those tools. | Claude Desktop, GitHub Copilot, or an AI agent inside WSO2 Integrator. |

WSO2 Integrator can play either role:

- **As a server** — turn an integration into a set of tools that any AI assistant can call.
- **As a client** — let your AI agents consume tools from any MCP server, including community servers for Slack, GitHub, file systems, and so on.

You can also do both at once: an integration that exposes some tools through MCP for outside assistants, and uses other MCP servers as part of its own AI agents.

## What Can an MCP Server Offer?

MCP defines three kinds of things a server can expose:

| Capability | What it is | Example |
|---|---|---|
| **Tools** | Functions the AI can call (with arguments, with a return value). | *"Look up customer by ID"*, *"Create a ticket"*. |
| **Resources** | Read-only data the AI can ask for. | A live snapshot of your knowledge base, a configuration file. |
| **Prompts** | Reusable prompt templates the AI can fill in. | A standard *"Onboard a new employee"* template. |

Most teams start with tools — they're the most useful and the easiest to wrap. Resources and prompts are powerful add-ons for richer integrations.

> **What WSO2 Integrator supports today:** Tools are fully supported, both for [exposing your integration as an MCP server](/docs/genai/develop/mcp/exposing-as-mcp) and for [consuming MCP servers from an agent](/docs/genai/develop/mcp/consuming-mcp-from-agent). Resources and Prompts are part of the MCP specification but are not yet documented for use in WSO2 Integrator.

## How a Conversation Looks Over MCP

The protocol itself is simple. Here is the rhythm of a typical interaction:

1. The client connects and asks the server: *"What can you do?"*
2. The server answers with the list of tools, their descriptions, and their parameter shapes.
3. When the AI needs something, the client says: *"Please run `lookupCustomer` with `customerId = 'C-1234'`"*.
4. The server runs the function and sends back the result.
5. The AI uses the result to answer the user — or to decide what to do next.

The AI assistant never sees the server's source code. It only sees the *shape* of each tool — the same way an [AI agent](what-is-ai-agent.md) sees its own tools. This is why writing clear descriptions on your MCP tools is just as important as writing clear descriptions on your local tools.

## Why It Matters for Integrations

The reason MCP is a big deal for an integration platform is that it standardises something that used to be one-off and brittle:

- **Reusability** — one server, many AI assistants.
- **Decoupling** — you can change the assistant without rebuilding the integration, and vice versa.
- **Ecosystem** — community-maintained MCP servers exist for many popular SaaS products. Your AI agent can plug straight in.
- **Governance** — because MCP is a clear boundary between "AI side" and "tools side", it's a natural place to put authentication, rate limiting, and auditing.

In short: MCP turns "AI plus your stuff" from a custom build into a configuration exercise.

## What MCP Is *Not*

A few common misconceptions:

- **MCP is not an LLM.** It does not generate text. It just connects AI assistants to tools.
- **MCP is not a replacement for your APIs.** It sits in front of them. The MCP server still calls your existing APIs, databases, and services.
- **MCP is not magic security.** Anything you expose over MCP becomes accessible to whoever connects. Treat your MCP server like any other public surface — authenticate, authorise, log.

## What's Next

- [What are Tools?](what-are-tools.md) — How tools work in general; MCP is one way to deliver them.
- [What is an AI Agent?](what-is-ai-agent.md) — The natural consumer of MCP servers.
- [Exposing a Service as MCP](/docs/genai/develop/mcp/exposing-as-mcp) — Expose your integrations as MCP tools.
- [Consuming MCP from an Agent](/docs/genai/develop/mcp/consuming-mcp-from-agent) — Use MCP tools from inside an agent.
