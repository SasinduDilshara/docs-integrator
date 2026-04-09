---
title: "What Is MCP?"
description: "Understand the Model Context Protocol -- a standard for connecting LLMs to external tools and services."
---

# What Is MCP?

The **Model Context Protocol (MCP)** is an open standard that gives Large Language Models a standardized way to discover, understand, and invoke external tools. It defines a common protocol so that any MCP client (an LLM, agent, or AI application) can connect to any MCP server (a service that exposes tools) without custom integration code.

MCP solves a fundamental interoperability problem in AI: without a standard protocol, every LLM application needs bespoke glue code to connect to each external tool. MCP eliminates this by providing a universal interface for tool discovery and invocation.

With WSO2 Integrator, you can both **expose your integrations as MCP servers** and **connect your agents to external MCP servers**, making your enterprise services available to any AI application that speaks the protocol.

## In This Section

- [Why MCP?](why-mcp.md) -- The problem MCP solves and the benefits of a standard tool protocol
- [How MCP Works](how-mcp-works.md) -- Core concepts, architecture, and transport options

## What's Next

After understanding MCP concepts, explore the hands-on guides:

- [What Is RAG?](../what-is-rag/index.md) -- Another key pattern for grounding LLM responses in your own data
- [What Are Tools?](../what-are-tools.md) -- How tools work within AI agents
