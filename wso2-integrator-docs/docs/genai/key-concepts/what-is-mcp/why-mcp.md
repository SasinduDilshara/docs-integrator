---
title: "Why MCP?"
description: "Understand the problem MCP solves and the benefits of a standardized tool protocol for AI."
---

# Why MCP?

## The Problem: Custom Glue Code Everywhere

Without MCP, connecting an LLM or AI agent to external tools requires custom integration code for every combination of AI application and tool provider. Each connection needs its own:

- Tool discovery logic (how does the AI know what tools are available?)
- Schema definitions (what parameters does each tool accept?)
- Invocation protocol (how do you call the tool and get results back?)
- Error handling (what happens when a tool call fails?)

This creates tight coupling between AI applications and tool providers. If you have 5 AI applications and 10 tool providers, you need up to 50 custom integrations. When an API changes, every integration that touches it breaks.

```
Without MCP:

App 1 ──custom code──► Tool A
App 1 ──custom code──► Tool B
App 2 ──custom code──► Tool A
App 2 ──custom code──► Tool B
App 3 ──custom code──► Tool A
...
```

## The Solution: A Standard Protocol

MCP introduces a single, standardized protocol that sits between AI applications and tool providers:

```
With MCP:

App 1 ──┐                ┌──► Tool A
App 2 ──┼── MCP protocol ──┼──► Tool B
App 3 ──┘                └──► Tool C
```

Any MCP client can connect to any MCP server. Build one MCP server for your service, and it works with Claude, ChatGPT, custom agents, and any other MCP-compatible application -- today and in the future.

## The "USB Port" Analogy

Think of MCP as the **USB port for AI**. Before USB, every peripheral device needed its own proprietary connector -- printers, keyboards, cameras, and storage devices all had different plugs and protocols. USB standardized the physical and logical connection, so any USB device works with any USB port.

MCP does the same for AI tools. Instead of building a proprietary connector for each AI-to-tool combination, you implement the MCP standard once on each side. The AI application speaks MCP as a client, the tool provider speaks MCP as a server, and they connect without any custom code.

## Benefits of MCP

### Interoperability

Once your service is exposed as an MCP server, it is usable by any MCP-compatible AI application. You do not need to rebuild integrations when you adopt a new LLM provider or AI framework.

### Reduced development effort

Instead of writing custom connector code for every AI application that needs your tools, you write one MCP server. Instead of writing custom tool-calling code for every external service, your agent connects via one MCP client.

### Standardized tool discovery

MCP clients can dynamically discover what tools a server offers, including their names, descriptions, parameter schemas, and return types. The AI application does not need hardcoded knowledge of available tools -- it learns them at runtime.

### Ecosystem growth

As more services adopt MCP, the ecosystem of available tools grows for everyone. A weather service, a database query tool, a document search system -- once each is exposed via MCP, any agent can use them without additional integration work.

### Future-proofing

Standards outlast individual implementations. By building on MCP, your integrations remain compatible as the AI landscape evolves, new models emerge, and new tools become available.

## What's Next

- [How MCP Works](how-mcp-works.md) -- Core concepts, client-server architecture, and transport options
