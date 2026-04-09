---
title: GenAI Overview
description: "Overview of GenAI capabilities in WSO2 Integrator and Ballerina for building AI-powered integrations."
slug: /genai
---

# GenAI Overview

WSO2 Integrator brings generative AI capabilities directly into your integration workflows. With the same visual and pro-code development experience you already use, you can build AI-powered integrations that go beyond simple API calls -- creating intelligent agents, grounding responses in your own data, and exposing your services as tools that any LLM can discover and use.

:::info What belongs here vs. Develop?
If AI is helping YOU code faster (Copilot, AI suggestions, AI-generated tests) -- that's in [Develop](/docs/develop). If YOU are building an AI-powered integration (agents, RAG, MCP) -- you're in the right place.
:::

## What GenAI in BI Enables

GenAI support in Ballerina Integrator lets you build AI-powered integrations that combine the reliability of traditional integration patterns with the flexibility of large language models. You can create services that understand natural language, reason over data, call tools autonomously, and adapt to user intent -- all within a governed, observable framework.

## Key Capabilities

- **Direct LLM Calls** -- Connect to OpenAI, Azure OpenAI, Anthropic, and other model providers to generate text, classify content, extract information, and more. See [Configuring LLM Providers](build-ai-applications/direct-llm-calls/configuring-llm-providers.md).

- **Natural Functions** -- Define typed function signatures and let the LLM execute logic described in plain language, bridging the gap between structured code and natural language processing. See [Defining Natural Functions](build-ai-applications/natural-functions/defining-natural-functions.md).

- **AI Agents** -- Create intelligent agents that perceive, reason, and act. Agents maintain conversational memory, call tools automatically, and can be exposed as HTTP endpoints. See [Creating an AI Agent](build-ai-applications/ai-agents/creating-an-ai-agent.md).

- **Model Context Protocol (MCP)** -- Expose your integrations as MCP-compatible tool servers that any LLM can discover and call, or consume tools from external MCP servers. See [MCP Integration](build-ai-applications/mcp-integration/exposing-services-as-mcp-server.md).

- **Retrieval-Augmented Generation (RAG)** -- Ground LLM responses in your own data by building ingestion pipelines, connecting to vector databases, and serving context-aware answers. See [RAG Ingestion](build-ai-applications/rag/rag-ingestion/chunking-strategies.md).

## Where to Go Next

- **Getting Started** -- Jump into a [Quick Start](getting-started/quick-start.md) to build a conversational agent in 10-15 minutes, or follow a step-by-step tutorial in [Build Your First AI Integration](getting-started/build-your-first-ai-integration/smart-calculator-assistant.md).

- **Key Concepts** -- Understand [What is an AI Agent?](key-concepts/what-is-an-ai-agent.md), [What are Tools?](key-concepts/what-are-tools.md), and [What is Memory?](key-concepts/what-is-memory.md).

- **Develop AI Applications** -- Explore guides for [AI Agents](build-ai-applications/ai-agents/creating-an-ai-agent.md), [MCP Integration](build-ai-applications/mcp-integration/exposing-services-as-mcp-server.md), [RAG](build-ai-applications/rag/rag-query/rag-querying.md), and [Agent Configuration Options](build-ai-applications/ai-agents/agent-configuration-options.md).

- **Reference** -- Review [AI Governance and Security](reference/ai-governance-and-security.md) and [Troubleshooting](reference/troubleshooting.md) for production deployments.
