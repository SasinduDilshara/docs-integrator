---
sidebar_position: 3
title: Troubleshooting and Common Issues
description: Solutions for common problems when building AI integrations with WSO2 Integrator.
---

# Troubleshooting and Common Issues

This page covers common issues encountered when building AI integrations with WSO2 Integrator using the `ballerina/ai` and `ballerina/mcp` modules, along with their solutions.

## Model Provider Issues

| Issue | Cause | Solution |
|---|---|---|
| `error: no model provider configured` when calling `ai:getDefaultModelProvider()` | There is no `[ballerina.ai]` block in `Config.toml`. | Run **"Configure default WSO2 Model Provider"** from the VS Code command palette, or add a provider block manually (for example, `[ballerinax.ai.openai]` with `apiKey = "..."`) and construct `new openai:ModelProvider(...)` yourself. |
| `401 Unauthorized` from the provider | Invalid or expired API key | Verify `openAiApiKey` (or the equivalent) in `Config.toml`; rotate if expired. |
| `429 Too Many Requests` | Rate limit exceeded | Apply per-session throttling in your service layer; consider upgrading your provider plan. |
| `Connection refused` to Ollama | Ollama not running | Start Ollama with `ollama serve` and verify the endpoint URL in `Config.toml`. |
| `Model not found` | Incorrect model identifier | Use the canonical constants exposed by the provider module, for example `openai:GPT_4O`. |
| `context_length_exceeded` | Input plus conversation history exceeds the model's context window | Shorten the system prompt, reduce stored conversation history, or move to a model with a larger context window. |

## Agent Issues

| Issue | Cause | Solution |
|---|---|---|
| `undefined symbol agent:ChatAgent` at compile time | Using the deprecated `ai.agent` module name. | Import `ballerina/ai` and use `ai:Agent`. There is no `ballerinax/ai.agent`. |
| Agent loops indefinitely | The agent cannot find the right tool or complete the task. | Tighten the system prompt. Make each tool's first doc-comment line a crisp instruction for *when* to use it. |
| Agent calls the wrong tool | Tool names or descriptions are ambiguous. | Rename tool functions to reflect the exact action and rewrite the doc comment to describe when to call the tool. |
| Agent does not call any tools | System prompt does not encourage tool use. | Explicitly tell the agent *when* to use each tool in the `instructions` field. |
| `isolated` compile error on a tool function | A tool function accesses non-`isolated` module state. | Mark the function `isolated` and confine shared state to `isolated` classes or `lock` blocks. |
| `@ai:AgentTool` parameter is ignored | You are trying to pass `{name: "...", description: "..."}` as annotation arguments. | `@ai:AgentTool` takes no arguments. Move the description into the function's doc comment; parameter descriptions live in `# + name - description` lines. |
| Session memory does not roll over between requests | The caller passes a different `sessionId` each time. | Use a stable `sessionId` per user and send it in every request via `ai:ChatReqMessage`. |
| Memory is lost after a service restart | You are relying on the in-process memory of `ai:Listener` or `ai:ShortTermMemory`. | Implement `ai:Memory` against a database (for example, PostgreSQL or Redis) and pass it to the agent via the `memory` field. |

## RAG Issues

| Issue | Cause | Solution |
|---|---|---|
| `check new ai:VectorKnowledgeBase(...)` fails to compile | `ai:VectorKnowledgeBase` is not fallible. | Use plain `new`: `new ai:VectorKnowledgeBase(vectorStore, embeddingProvider)`. |
| No results from `knowledgeBase.retrieve(...)` | Embedding provider or vector store mismatch, or the knowledge base was never ingested. | Call `ingest(...)` at least once, and make sure `ai:getDefaultEmbeddingProvider()` is configured. |
| Irrelevant results | Chunks too large or too small, or the wrong embedding model for the domain. | Tune your `ai:DataLoader` chunk size and experiment with different embedding providers. |
| Slow retrieval at scale | Using `ai:InMemoryVectorStore` for large corpora. | Switch to `ballerinax/ai.pinecone`, `ballerinax/ai.weaviate`, `ballerinax/ai.milvus`, or `ballerinax/ai.pgvector` — the rest of your code does not change. |
| Empty responses from the RAG tool | You forgot to call `ai:augmentUserQuery` before `model->chat(...)`. | Always wrap the retrieved chunks in `ai:augmentUserQuery(context, question)` before sending them to the model. |

## MCP Issues

| Issue | Cause | Solution |
|---|---|---|
| `module ballerinax/mcp not found` | Wrong module org. | The module is `ballerina/mcp` and ships with the distribution. Remove any `ballerinax/mcp` import or `[[dependency]]` entry. |
| `ai:McpToolKit` fails to connect | MCP server is not running, or the URL does not end with `/mcp`. | Start the MCP server and double-check the URL — a `ballerina/mcp` service mounted on `/mcp` will respond at `http://host:port/mcp`. |
| MCP tools do not appear to the agent | The tool kit was created but not passed in the `tools` array. | Add the `ai:McpToolKit` instance to the agent's `tools = [...]`. |
| MCP tool has no description | The remote function is missing a doc comment. | Add a `#`-style doc comment above the `remote function`. The first line becomes the tool description; parameter descriptions come from `# + name - description` lines. |
| `natural expression is an experimental feature` | You are using the natural-expression syntax on a Ballerina distribution that requires the experimental flag. | Run with `bal run --experimental`. |

## Build and Runtime Errors

| Issue | Cause | Solution |
|---|---|---|
| `module not found: ballerinax/ai.agent` | Using the deprecated module name. | Replace `import ballerinax/ai.agent;` with `import ballerina/ai;`. Remove any `[[dependency]]` entry for `ai.agent`. |
| `module not found: ballerinax/ai.rag` | RAG types are in `ballerina/ai`. | Use `import ballerina/ai;` and reference `ai:VectorKnowledgeBase`, `ai:InMemoryVectorStore`, `ai:TextDataLoader`, and friends from there. |
| `module not found: ballerinax/ai.provider.openai` | Wrong provider path. | Use `import ballerinax/ai.openai;` and `openai:ModelProvider`. |
| `Incompatible types` in tool functions | Tool return type does not match its JSON shape. | Declare a specific record type as the return and let the type system validate LLM-produced JSON. |
| `Config variable not found` | Missing `Config.toml` entry. | Ensure every `configurable` variable is present either in `Config.toml` or through a `BAL_CONFIG_VAR_*` environment variable. |

## Getting Help

- Check the [WSO2 Community Forums](https://community.wso2.com) for community support.
- File issues on the [WSO2 Integrator GitHub repository](https://github.com/wso2/integrator).
- Review the [Ballerina AI module documentation](https://central.ballerina.io/ballerina/ai/latest).
- Review the [Ballerina MCP module documentation](https://central.ballerina.io/ballerina/mcp/latest).

## What's Next

- [AI Governance and Security](ai-governance.md) — Compliance and data handling
- [Ballerina Copilot Guide](copilot-guide.md) — AI-assisted development
