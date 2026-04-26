---
sidebar_position: 3
title: Tools
description: Reference for adding tools to a WSO2 Integrator AI agent — connections, functions, MCP servers, and custom tools.
---

# Tools

Tools are the buttons your AI Agent can press. WSO2 Integrator gives you four ways to add a tool, all from the same dialog. This page is the reference for that dialog and what each option does.

## The Add Tool Dialog

On the agent canvas, with the AI Agent node selected, click **+ Add Tool** (or use the right-side **Tools** panel). The **Add Tool** dialog lists the four kinds of tool you can add:

![The Add Tool dialog with four options: Use Connection, Use Function, Use MCP Server, Create Custom Tool. Each option has a one-line description.](/img/genai/develop/agents/05-add-tool-dialog.png)

| Option | Use when… |
|---|---|
| **Use Connection** | You want the agent to use one of WSO2 Integrator's connectors (Salesforce, Gmail, MySQL, GitHub, …). Each operation of the connector becomes a tool. |
| **Use Function** | You already have a function in your project, or in the standard library, that the agent should be able to call. |
| **Use MCP Server** | The tools live on a remote MCP server (your own, a community server, or a SaaS MCP endpoint). |
| **Create Custom Tool** | You want to define a new tool from scratch — name, description, parameters, return — without writing code first. |

Each option opens a different second-step panel. The four sections below describe them.

## 1. Use Connection — Connector-Backed Tools

Picking **Use Connection** opens the connection picker. You can either:

- Pick an existing connection from the project tree.
- Add a new connection. The picker lists pre-built connectors (HTTP, GraphQL, WebSocket, TCP, UDP, FTP, MySQL, MongoDB, PostgreSQL, MS SQL, Redis, Kafka Consumer, …) and offers two import paths:
  - **Connect via API Specification** — import an OpenAPI or WSDL file to generate a connector on the fly.
  - **Connect to a Database** — enter credentials and let BI introspect tables.

After selecting the connection, BI lists its operations. Tick the operations you want the agent to be able to call. Each ticked operation becomes a tool — its description and parameter shapes come from the connector's metadata.

Use this when an off-the-shelf connector already does what you need; you save writing wrapper code.

## 2. Use Function — Project or Library Functions

Picking **Use Function** opens the **Create Tool from Function** panel:

![The Create Tool from Function panel showing two sections: Within Project (functions defined in the current project) and Standard Library, with subgroups for io (fileReadJson, fileReadString, fileWriteJson, fileWriteString, print, println), log (printDebug, printError, printInfo, printWarn), and time (utcFromString, utcNow). Imported Functions section at the bottom.](/img/genai/develop/agents/06-tool-from-function.png)

Three kinds of functions can be turned into tools:

| Group | Notes |
|---|---|
| **Within Project** | Any function in your own code, including [natural functions](/docs/genai/develop/natural-functions/overview), can become a tool with one click. |
| **Standard Library** | Common Ballerina utilities — `io:fileReadJson`, `log:printInfo`, `time:utcNow`, etc. |
| **Imported Functions** | Functions from any module already imported in your project. |

Pick a function and BI:

- Reads its **doc comment** as the tool description.
- Reads each **parameter description** (the `# + name - description` lines) as parameter metadata for the LLM.
- Reads the **return type** as the tool's output schema.
- Adds the `@ai:AgentTool` annotation if the function doesn't have one yet.

If a function lacks a doc comment, BI prompts you to add one — the LLM cannot use a tool with no description. The first sentence of the doc comment is what the LLM sees, so make it specific.

> **Tip:** Natural functions make excellent tools. The agent's main LLM handles routing; the natural function uses (potentially) a different model with a tighter prompt and stricter return type for the actual subtask.

## 3. Use MCP Server — Tools Over the Standard Protocol

Picking **Use MCP Server** opens the **Add MCP Server** panel:

![The Add MCP Server panel — Tools to Include set to All, then Advanced Configurations including Info (name, version), HTTP Version with a Select / Expression toggle, HTTP1 Settings, HTTP2 Settings, Timeout (default 30 seconds), Forwarded.](/img/genai/develop/agents/08-add-mcp-server.png)

Configuration:

| Field | What it does |
|---|---|
| **Server URL** (top of the panel, scrolled out of frame) | The MCP endpoint, for example `http://localhost:9090/mcp` or `https://mcp.example.com`. |
| **Tools to Include** | `All` exposes every tool the server advertises; or pick a subset by name. |
| **Info → name / version** | Identification of the MCP client your agent presents. Defaults are usually fine. |
| **HTTP Version** | `HTTP/2_0` for modern Streamable HTTP; `HTTP/1_1` for older servers. |
| **HTTP1 / HTTP2 Settings** | Protocol-specific tuning — keep-alive, header sizes, frame sizes. |
| **Timeout** | Per-call timeout. The default is 30 seconds. Increase for slow tools. |
| **Forwarded** | Whether to set the `Forwarded` / `X-Forwarded-For` header when this MCP client sits behind a proxy. |

After saving, every tool the MCP server advertises (or every tool in your **Tools to Include** list) is now available to the agent. They appear in the agent's tool list alongside any local function tools, indistinguishable from the agent's perspective.

Common uses:

- A WSO2 Integrator project consuming **its own** MCP service (see [Exposing a Service as MCP](/docs/genai/develop/mcp/exposing-as-mcp)).
- A community MCP server for Slack, GitHub, file systems, or your CRM.
- A SaaS MCP endpoint your vendor provides.

> **Tip:** Filter to the specific tools you need. LLMs choose better when the tool list is short and specific.

## 4. Create Custom Tool — Hand-Crafted Definitions

Picking **Create Custom Tool** opens the **Create New Agent Tool** dialog:

![The Create New Agent Tool dialog. Name field set to CustomTool. Description text area for explaining when the agent should use the tool. Parameters with an Add Parameter link. Return Type field. Description for the return type. Advanced Configurations Expand link. Create button at the bottom.](/img/genai/develop/agents/07-create-custom-tool.png)

Use this when you want to define a tool *before* you write its implementation, or when the tool's shape is genuinely custom.

| Field | Required | What it does |
|---|---|---|
| **Name** | Yes | The tool's name. The LLM uses this to refer to it. Camel-case, descriptive: `lookupCustomer`, `bookAppointment`. |
| **Description** | Yes | What the tool does **and** when the agent should use it. The single most important field for tool quality. |
| **Parameters** | No | Each parameter has a name, type, and description. The description tells the LLM what to put in. |
| **Return Type** | Yes | The Ballerina type of the return value. Drives the schema the LLM sees. |
| **Description** (return) | No | One-line description of the return value, included in the schema. |
| **Advanced Configurations** | No | Visibility, isolation requirements, retries. Defaults are usually fine. |

After you click **Create**, BI generates a stub function in your project. You fill in the implementation — call an API, run a query, do whatever the tool is supposed to do — and the agent can already select it.

## After Adding a Tool

The new tool shows up in the agent's right-side **Tools** panel and is included in every reasoning step from then on. Two things to keep in mind:

- **Mention the tool in Instructions.** A line in the system prompt that says *"Use `lookupCustomer` whenever the user mentions a customer ID"* significantly improves tool-selection accuracy.
- **Watch the count.** Above ~10 tools, models start picking less reliably. Group related tools into a [toolkit](#toolkits) instead, or split the agent into a router agent plus specialists.

## Toolkits

A **toolkit** is a class that bundles related tools — for example, a `BillingToolkit` that exposes `getInvoice`, `processRefund`, and `getPaymentHistory`. Toolkits are written in source view today (no separate UI yet); you create one Ballerina class with the tool methods and register the toolkit in the agent's tool list. The agent treats the toolkit's methods exactly like individual tools.

Use toolkits when:

- A group of tools shares state (a connector, a database client, a configuration value).
- You want to enable / disable a set of related tools as a unit (read-only mode vs full access).

## Tool Quality Checklist

Run through this before shipping an agent:

- [ ] Each tool has a one-sentence description that names *what* and *when*.
- [ ] Each parameter has a description.
- [ ] The return type is the smallest one that contains what the agent needs (no kitchen-sink JSON).
- [ ] Destructive tools mention the confirmation requirement in their description.
- [ ] The agent's Instructions reference each tool at least once.
- [ ] You've tried the agent with edge cases — empty inputs, ambiguous questions, off-topic requests.

## What's Next

- **[Memory](memory.md)** — make the agent's tool calls remember earlier turns.
- **[Observability](observability.md)** — see which tools the agent actually picks.
- **[What are Tools?](/docs/genai/key-concepts/what-are-tools)** — conceptual background.
