---
sidebar_position: 1
title: Overview
description: A quick orientation to the WSO2 Integrator BI workspace for building AI features, the five AI building blocks and the AI side panel where you create them.
---

# Develop AI Applications Overview

Before you dive into a specific feature, two things are worth knowing up front: **what AI building blocks BI gives you**, and **where in the UI you create them**. That's all this page covers. The feature pages that follow assume you know both.

## The Five AI Building Blocks

Everything AI-related in BI boils down to these five blocks. Pick the one that matches your task, the rest of this section is a reference for each.

| Block | Use it when you want to… | Reference |
|---|---|---|
| **Direct LLM Call** | Send one prompt to an LLM and bind the response to a typed variable inside a flow. The simplest, no-frills entry point. | [Direct LLM Calls](/docs/genai/develop/direct-llm/overview) |
| **Natural Function** | Define a typed function whose body is plain English. Reuse the same prompt across the codebase like any other function. | [Natural Functions](/docs/genai/develop/natural-functions/overview) |
| **RAG** | Ground LLM responses in your own documents, load data, chunk and embed it, store it in a vector store, and retrieve relevant context at query time. | [RAG](/docs/genai/develop/rag/overview) |
| **AI Agent** | Combine an LLM with tools and memory so it can reason over multiple turns, call external systems, and produce a final answer. Exposed as a service. | [AI Agents](/docs/genai/develop/agents/overview) |
| **MCP** | Either expose your project's functions as MCP tools for any MCP client, or consume external MCP servers from inside an agent. | [MCP Integration](/docs/genai/develop/mcp/overview) |

A rough mental model: **Direct LLM** is one call, **Natural Function** is a reusable prompt, **RAG** adds your data, **Agent** adds reasoning and tools, **MCP** is the protocol for sharing tools across systems.

## The Artifacts Panel

Two of the five blocks, **AI Chat Agent** and **MCP Service**, are top-level services, not nodes inside a flow. You create them from the **Artifacts** panel: open it from the integration project view and pick from the **AI Integration** group.

![The Artifacts panel with sections for Automation, AI Integration (AI Chat Agent and MCP Service), Integration as API (HTTP Service, GraphQL Service, TCP Service), Event Integration (Kafka, RabbitMQ, MQTT, Azure Service Bus, Salesforce, Twilio, GitHub, Solace, CDC for Microsoft SQL Server, CDC for PostgreSQL), and File Integration.](/img/genai/develop/shared/07-artifacts-page-full.png)

| AI Integration artifact | What you get |
|---|---|
| **AI Chat Agent** | A complete chat-agent service, listener, agent node, return, opened on the canvas ready for instructions, tools, memory, and a model provider. |
| **MCP Service** | An MCP server artifact that exposes tools to any MCP client. Add tools to it the same way you add resources to an HTTP service. |

The other three blocks (**Direct LLM**, **RAG**, **Natural Function**) aren't artifacts, they're things you add inside an existing flow, which is where the side panel comes in.

## The AI Side Panel

Every AI block above is added the same way: open a flow, click the **+** between two nodes, and the **Add Node** side panel slides in from the right. The **AI** category has three sub-categories that map directly to the building blocks.

![The Add Node panel with categories: Statement, Control, AI (with three sub-categories, Direct LLM, RAG, Agent), Error Handling, Concurrency, Logging.](/img/genai/develop/shared/04-add-node-panel-full.png)

| AI sub-category | What it gives you |
|---|---|
| **Direct LLM** | The **Model Provider** node, adds a connection to an LLM (OpenAI, Anthropic, Azure, WSO2 Default, …) and exposes its `Generate` and `Chat` actions. |
| **RAG** | **Knowledge Base**, **Data Loader**, and **Augment Query** nodes, the three pieces of a RAG pipeline. |
| **Agent** | The **Agent** node, drops a complete agent block into the flow. For most use cases the **AI Chat Agent** artifact (Add Artifact → AI Integration) is the easier starting point. |

Pick a node, fill the wizard that opens, and BI generates the matching Ballerina source. You don't write imports or call signatures by hand, but you can always switch to source view if you want to read or edit what was generated.

That's the whole orientation. Head into any feature page below, they all follow this same create-via-side-panel flow.
