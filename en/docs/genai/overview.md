---
title: AI Integrations Overview
sidebar_label: Overview
slug: /genai/overview
---

# AI Integrations

Build AI-powered integrations with WSO2 Integrator — direct LLM calls, natural functions, RAG pipelines, AI agents, and MCP servers — all from a single `ballerina/ai` abstraction that works with any LLM provider.

## Getting Started

- **[Setting Up WSO2 Integrator](getting-started/setup.md)** — Install and configure your first AI-ready project
- **[Build a Smart Calculator Assistant](getting-started/smart-calculator.md)** — Your first agent with tool calling
- **[Build a Sample Hotel Booking Agent](getting-started/hotel-booking-agent.md)** — Conversational agent with memory and a chat endpoint

## Key Concepts

- **[What is an LLM?](key-concepts/what-is-llm.md)**
- **[What is a Natural Function?](key-concepts/what-is-natural-function.md)**
- **[What is an AI Agent?](key-concepts/what-is-ai-agent.md)**
- **[What are Tools?](key-concepts/what-are-tools.md)**
- **[What is AI Agent Memory?](key-concepts/what-is-agent-memory.md)**
- **[What is MCP?](key-concepts/what-is-mcp.md)**
- **[What is RAG?](key-concepts/what-is-rag.md)**

## Develop AI Applications

- **[Direct LLM Calls](develop/direct-llm.md)** — Call any LLM through `ai:ModelProvider`
- **[Natural Functions](develop/natural-functions/defining.md)** — Replace hand-written logic with typed `natural { ... }` expressions
- **[RAG](develop/rag/chunking-documents.md)** — Ingest documents into a knowledge base and query them with grounded context
- **[AI Agents](develop/agents/creating-agent.md)** — Build agents that reason over tools, memory, and external systems
- **[MCP Integration](develop/mcp/creating-mcp-server.md)** — Expose Ballerina services as MCP tools, or consume MCP servers from an agent

## Tutorials

- **[HR Knowledge Base Agent with RAG](tutorials/hr-knowledge-base-rag.md)**
- **[Customer Care Agent with MCP](tutorials/customer-care-mcp.md)**
- **[IT Helpdesk Chatbot with Persistent Memory](tutorials/it-helpdesk-chatbot.md)**
- **[Legal Document Q&A with MCP and RAG](tutorials/legal-doc-qa.md)**

## Reference

- **[Ballerina Copilot Setup and Usage](reference/copilot-guide.md)**
- **[AI Governance and Security](reference/ai-governance.md)**
- **[Troubleshooting and Common Issues](reference/troubleshooting.md)**
