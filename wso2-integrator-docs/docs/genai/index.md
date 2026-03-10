---
sidebar_position: 4
title: GenAI
description: "How do I build AI agents, RAG apps, or MCP servers?"
---

# GenAI

Build AI-powered integrations with WSO2 Integrator. Create intelligent agents, implement RAG pipelines, expose MCP servers, and add guardrails — all with the same visual + pro-code development experience.

:::info What belongs here vs. Develop?
If AI is helping YOU code faster (Copilot, AI suggestions, AI-generated tests) → that's in [Develop](/docs/develop). If YOU are building an AI-powered integration (agents, RAG, MCP) → you're in the right place.
:::

## Quick Starts

Jump in with a focused, 10–15 minute walkthrough. These assume you've completed [Set Up](/docs/get-started/install).

- [Build a Conversational Agent](quick-starts/conversational-agent.md) — Create an agent with memory, bind tools, test via chat
- [Build a RAG Application](quick-starts/rag-application.md) — Ingest documents, embed, query with LLM augmentation
- [Expose an MCP Server](quick-starts/mcp-server.md) — Turn an integration into an MCP-compatible tool server

## Agents

Create intelligent agents that perceive, reason, and act — powered by LLMs and connected to your existing services.

- [Agent Architecture & Concepts](agents/architecture-concepts.md) — The agent loop, memory types, and how agents differ from simple LLM calls
- [Chat Agents](agents/chat-agents.md) — Build user-facing conversational agents with system prompts and streaming
- [API-Exposed Agents](agents/api-exposed-agents.md) — Expose agents as HTTP/REST endpoints for programmatic access
- [Agent Memory Configuration](agents/memory-configuration.md) — Conversation, semantic, and persistent memory options
- [Tool Binding (Function Calling)](agents/tool-binding.md) — Connect agents to external services, connectors, and APIs
- [Multi-Agent Orchestration](agents/multi-agent-orchestration.md) — Route between specialized agents with supervisor patterns
- [Natural Functions](agents/natural-functions.md) — Describe logic in plain language and let LLMs execute it at runtime

## RAG (Retrieval-Augmented Generation)

Ground LLM responses in your own data. Build ingestion pipelines, connect to vector databases, and serve context-aware answers.

- [RAG Architecture Overview](rag/architecture-overview.md) — Indexing and query phases, when to use RAG vs. fine-tuning
- [Vector Database Connectivity](rag/vector-database.md) — Connect to Pinecone, Weaviate, Chroma, pgvector, and more
- [Document Ingestion Pipelines](rag/ingestion-pipelines.md) — Load, parse, chunk, embed, and store documents
- [Chunking & Embedding Strategies](rag/chunking-embedding.md) — Choose chunking methods and embedding models
- [Building a RAG Service](rag/building-rag-service.md) — Create an HTTP service that answers questions from your knowledge base

## MCP (Model Context Protocol)

Expose your integrations as tools that any LLM can discover and call, or consume tools from external MCP servers.

- [MCP Overview](mcp/overview.md) — What MCP is and why it matters for integrations
- [Exposing Integrations as MCP Servers](mcp/exposing-mcp-servers.md) — Turn services into MCP-compatible tool providers
- [Consuming MCP Tools](mcp/consuming-mcp-tools.md) — Connect agents to external MCP servers
- [MCP Security & Authentication](mcp/security-authentication.md) — Secure endpoints, rate limiting, and audit logging

## LLM Connectivity

Work with large language models effectively — choose models, craft prompts, handle streaming, and manage context.

- [Model Selection Guide](llm-connectivity/model-selection.md) — Compare models for different integration tasks
- [Prompt Engineering for Integrations](llm-connectivity/prompt-engineering.md) — System prompts, few-shot patterns, and output formatting
- [Streaming Responses](llm-connectivity/streaming-responses.md) — Configure streaming from LLMs in your services
- [Managing Context Windows](llm-connectivity/context-windows.md) — Token counting, truncation, and summarization strategies

## Guardrails & Safety

Protect your AI integrations with input validation, content filtering, cost controls, and responsible AI practices.

- [Input / Output Guardrails](guardrails/input-output-guardrails.md) — Validate inputs before LLMs and outputs before users
- [Content Filtering](guardrails/content-filtering.md) — Block harmful content, detect PII, enforce topic restrictions
- [Token & Cost Management](guardrails/token-cost-management.md) — Track usage, set limits, and optimize for cost
- [Responsible AI Practices](guardrails/responsible-ai.md) — Transparency, bias mitigation, and human-in-the-loop patterns

## Agent Observability

Monitor, trace, and debug your AI agents in development and production.

- [Agent Tracing](observability/agent-tracing.md) — End-to-end traces of agent execution with OpenTelemetry
- [Conversation Logging](observability/conversation-logging.md) — Structured logs for agent interactions
- [Performance Metrics](observability/performance-metrics.md) — Latency, token usage, error rates, and dashboards
- [Debugging Agent Behavior](observability/debugging-agent-behavior.md) — Diagnose common failure modes and inspect agent reasoning

## GenAI Tutorials

End-to-end walkthroughs that combine multiple GenAI capabilities into real-world scenarios (30–45 minutes each).

- [AI-Powered Customer Support Agent](tutorials/customer-support-agent.md) — Agent with order lookup, FAQ retrieval (RAG), and ticket creation
- [RAG-Powered Knowledge Base](tutorials/rag-knowledge-base.md) — Document ingestion pipeline + query service + chat interface
- [Multi-Agent Workflow Orchestration](tutorials/multi-agent-workflow.md) — Supervisor agent routing to billing, shipping, and technical specialists
- [MCP Server for Enterprise Data](tutorials/mcp-enterprise-data.md) — Expose database/API as MCP tools for any LLM to consume
- [Conversational Data Pipeline](tutorials/conversational-data-pipeline.md) — Agent that queries databases, transforms data, and generates reports
