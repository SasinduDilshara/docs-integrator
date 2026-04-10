---
sidebar_position: 1
title: Ballerina Copilot Setup and Usage Guide
description: Set up and use Ballerina Copilot for AI-assisted integration development in WSO2 Integrator.
---

# Ballerina Copilot Setup and Usage Guide

Ballerina Copilot is the AI-powered coding assistant that ships with the WSO2 Integrator extension for VS Code. It helps you write integration code faster by suggesting completions inline, generating Ballerina code from natural-language descriptions, and configuring the default WSO2 Model Provider for your project.

:::info This page is about AI helping YOU code
Ballerina Copilot assists you while you write code. For building AI-powered integrations (AI Agents, RAG, MCP), see the [GenAI develop guides](/docs/genai/develop/direct-llm).
:::

## Prerequisites

- VS Code with the WSO2 Integrator extension installed
- An active WSO2 account with Copilot access
- Internet connection for AI model access

## Signing In

All Copilot actions are available from the VS Code command palette (`Cmd+Shift+P` on macOS, `Ctrl+Shift+P` on Windows and Linux).

1. Open the command palette.
2. Run **"WSO2 Copilot: Sign In"**.
3. A browser window opens — complete the sign-in with your WSO2 account.
4. Once signed in, inline code completion and suggestions are enabled automatically.

You can verify your status with **"WSO2 Copilot: Show Status"** and sign out at any time with **"WSO2 Copilot: Sign Out"**.

## Key Commands

The following commands are the main entry points to Ballerina Copilot:

| Command | What it does |
|---|---|
| **WSO2 Copilot: Sign In** | Authenticates your VS Code session with WSO2. |
| **WSO2 Copilot: Generate Code** | Generates Ballerina code from a natural-language description. |
| **Configure default WSO2 Model Provider** | Writes a `[ballerina.ai]` block into your project's `Config.toml` so that `ai:getDefaultModelProvider()` works without your own API key. |

## Configuring the Default Model Provider

When you build an AI integration in Ballerina, code typically calls `ai:getDefaultModelProvider()`:

```ballerina
import ballerina/ai;

final ai:ModelProvider model = check ai:getDefaultModelProvider();
```

For this call to succeed, the runtime needs to know which provider to use. The easiest way is to let Copilot configure it for you:

1. Open the command palette.
2. Run **"Configure default WSO2 Model Provider"**.
3. Select the provider (for example, the WSO2 default, Azure OpenAI, or OpenAI).
4. The command updates your project's `Config.toml` with a `[ballerina.ai]` block — for the WSO2 default, it also provisions an access token.

After this, `ai:getDefaultModelProvider()` and `ai:getDefaultEmbeddingProvider()` will both work without you writing any additional configuration.

:::tip Bring your own key instead
If you would rather use your own provider key, skip the configure command and define a `configurable` value plus a provider from `ballerinax/ai.openai` (or similar):

```ballerina
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:ModelProvider model = check new openai:ModelProvider(
    openAiApiKey,
    modelType = openai:GPT_4O
);
```
:::

## Generating Code from Natural Language

Use **"WSO2 Copilot: Generate Code"** to describe what you want and let Copilot produce a complete Ballerina snippet. For example, if you describe:

> "Create an HTTP service that validates JSON payloads and stores them in a PostgreSQL database."

Copilot generates a complete service, including the database client, the resource method, and the validation logic. Review the result before accepting it.

## Inline Suggestions

Once you are signed in, Copilot automatically offers inline suggestions as you type in any `.bal` file. Press `Tab` to accept a suggestion, `Esc` to dismiss, or continue typing to ignore it.

Copilot considers your current file, your imports, and your project structure as context for its suggestions. If suggestions feel off, improve the context by adding the right imports and type definitions at the top of your file.

## Privacy and Data Handling

- Code context is sent to the WSO2 Copilot service for processing.
- No code is stored permanently on WSO2 servers.
- You can disable inline suggestions in VS Code settings under the WSO2 Integrator extension.
- For data-sensitive projects, use the WSO2 default provider so calls stay inside your WSO2 tenancy, or bring your own provider and configure a private endpoint.

## Troubleshooting

| Issue | Solution |
|---|---|
| No suggestions appearing | Run **"WSO2 Copilot: Sign In"** from the command palette and confirm your session is active. |
| `ai:getDefaultModelProvider()` returns an error | Run **"Configure default WSO2 Model Provider"** or add a `[ballerina.ai]` block to `Config.toml` manually. |
| Generated code uses unknown imports | Keep your Ballerina distribution up to date — the `ballerina/ai` and `ballerina/mcp` modules evolve quickly. |
| Authentication errors | Sign out (**"WSO2 Copilot: Sign Out"**) and sign in again. |

## What's Next

- [AI Governance and Security](ai-governance.md) — Data handling and compliance
- [Troubleshooting](troubleshooting.md) — Common issues and solutions
