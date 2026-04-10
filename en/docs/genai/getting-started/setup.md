---
sidebar_position: 1
title: Setting Up WSO2 Integrator for AI
description: Install and configure WSO2 Integrator to build AI-powered integrations with the Ballerina AI module.
---

# Setting Up WSO2 Integrator for AI

Before building AI integrations, you need the WSO2 Integrator development environment set up with access to an LLM. This page walks you through installation, project creation, and configuring a model provider so you can start building agents, RAG pipelines, and MCP servers.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- Ballerina Swan Lake Update 13 (`2201.13.x`) or newer — bundled with WSO2 Integrator
- One of:
  - A **WSO2 Copilot** account (recommended — gives you a ready-to-use default model provider, no third-party API key required), or
  - An API key for OpenAI, Anthropic, Azure OpenAI, Mistral, DeepSeek, or a local Ollama installation

The Ballerina `ai` module and MCP module ship with the distribution — you do not need to add them to `Ballerina.toml` manually.

## Step 1: Create a New Integration Project

Open VS Code with the WSO2 Integrator extension and create a new project, or use the CLI:

```bash
bal new my_ai_project
cd my_ai_project
```

This creates a standard Ballerina package with a `Ballerina.toml` and a `main.bal`.

## Step 2: Choose a Model Provider

The `ballerina/ai` module provides a single abstraction — `ai:ModelProvider` — that you use to talk to any LLM. You pick **one** of the two approaches below.

### Option A: WSO2 default model provider (recommended)

WSO2 Integrator ships with a default model provider that uses your WSO2 Copilot credentials. No separate API key is required.

1. In VS Code, open the command palette (`Cmd/Ctrl + Shift + P`).
2. Run **"Configure default WSO2 Model Provider"**.
3. Sign in when prompted. VS Code writes the configuration into your project's `Config.toml` automatically.

You can then get a model provider anywhere in your code with:

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();
```

This is the pattern used throughout the WSO2 Integrator AI documentation.

### Option B: Bring your own LLM provider

If you want to call OpenAI, Anthropic, Azure OpenAI, or another provider directly, add the corresponding connector module and initialize a provider explicitly.

#### OpenAI

```ballerina
import ballerina/ai;
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:ModelProvider model = check new openai:ModelProvider(
    openAiApiKey,
    modelType = openai:GPT_4O
);
```

```toml
# Config.toml
openAiApiKey = "sk-your-api-key-here"
```

#### Anthropic

```ballerina
import ballerina/ai;
import ballerinax/ai.anthropic;

configurable string anthropicApiKey = ?;

final ai:ModelProvider model = check new anthropic:ModelProvider(
    anthropicApiKey,
    anthropic:CLAUDE_SONNET_4_20250514,
    "2023-06-01"
);
```

```toml
# Config.toml
anthropicApiKey = "sk-ant-your-key-here"
```

#### Azure OpenAI

```ballerina
import ballerina/ai;
import ballerinax/ai.azure;

configurable string azureServiceUrl = ?;
configurable string azureApiKey = ?;
configurable string azureDeploymentId = ?;

final ai:ModelProvider model = check new azure:OpenAiModelProvider(
    azureServiceUrl,
    azureApiKey,
    azureDeploymentId,
    "2023-07-01-preview"
);
```

```toml
# Config.toml
azureServiceUrl   = "https://your-resource.openai.azure.com"
azureApiKey       = "your-azure-key"
azureDeploymentId = "your-deployment-id"
```

#### Other providers

The following connector modules follow the same pattern — import, declare a `configurable` for the API key, and construct the provider:

| Provider  | Module                    |
| --------- | ------------------------- |
| Mistral   | `ballerinax/ai.mistral`   |
| DeepSeek  | `ballerinax/ai.deepseek`  |
| Ollama    | `ballerinax/ai.ollama`    |

:::tip
Because every provider returns an `ai:ModelProvider`, the rest of your code — agents, natural expressions, RAG — never needs to change if you swap providers.
:::

## Step 3: Verify the Setup

Create a minimal test that asks the model to generate a response. Replace `main.bal` with the following:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:ModelProvider model = check ai:getDefaultModelProvider();

public function main() returns error? {
    string response = check model->generate(`Say "Setup complete!" in one sentence.`);
    io:println(response);
}
```

Run the project:

```bash
bal run
```

If you see a response from the LLM, your setup is complete.

:::info
If you are using a natural expression (`natural { ... }`) later, it is still an experimental feature. Run with `bal run --experimental` to enable it.
:::

## What's Next

- [Build a Smart Calculator Assistant](smart-calculator.md) — Your first AI integration using tool calling
- [Build a Sample Hotel Booking Agent](hotel-booking-agent.md) — An agent with memory and multiple tools
- [Key Concepts: What is an LLM?](/docs/genai/key-concepts/what-is-llm) — Understand the foundational technology
