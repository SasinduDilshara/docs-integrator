---
sidebar_position: 3
title: Consuming MCP from an Agent
description: Reference for using MCP servers as tool sources for WSO2 Integrator AI agents — the Add MCP Server panel, filters, and combining with local tools.
---

# Consuming MCP from an Agent

An [AI Agent](/docs/genai/develop/agents/overview) in WSO2 Integrator can use any MCP server as a tool source. Add the server once; the agent picks up every tool the server advertises (or only the ones you select) and treats them exactly like local function tools.

## The Add MCP Server Panel

On the agent canvas, click **+ Add Tool** → **Use MCP Server**. The **Add MCP Server** panel opens:

![The Add MCP Server panel. Tools to Include is set to All. Advanced Configurations expanded showing: Info (name and version), HTTP Version with Select / Expression toggle, HTTP1 Settings, HTTP2 Settings, Timeout (default 30 seconds), Forwarded.](/img/genai/develop/agents/08-add-mcp-server.png)

| Field | What it does |
|---|---|
| **Server URL** | The MCP endpoint, for example `http://localhost:9090/mcp`. |
| **Tools to Include** | `All` to expose every tool the server advertises, or a list of names to expose only some. |
| **Info → name** | The MCP client identifier sent to the server. |
| **Info → version** | A version string for your client. |
| **HTTP Version** | `HTTP/2_0` for modern Streamable HTTP servers; `HTTP/1_1` for older ones. |
| **HTTP1 / HTTP2 Settings** | Protocol tuning — keep-alive, header sizes, frame sizes. |
| **Timeout** | Per-call timeout. Default 30 s; increase for slow tools. |
| **Forwarded** | Whether to send `Forwarded` / `X-Forwarded-For` when sitting behind a proxy. |

After **Save**, every selected tool from the MCP server appears in the agent's tool list. They are invisible to the agent's LLM as "MCP tools" specifically — they look like any other tool.

## What BI Generates

Behind the panel, BI writes one connection line:

```ballerina
import ballerina/ai;

final ai:McpToolKit weatherMcp = check new ("http://localhost:9090/mcp");
```

Filter to specific tools:

```ballerina
final ai:McpToolKit readOnlyWeather = check new (
    "http://localhost:9090/mcp",
    ["getCurrentWeather", "getWeatherForecast"]
);
```

The toolkit goes into the agent's `tools = [...]` list:

```ballerina
final ai:Agent weatherAgent = check new (
    systemPrompt = {
        role: "Weather-aware Assistant",
        instructions: "You help users plan based on accurate weather information."
    },
    tools = [weatherMcp],
    model = check ai:getDefaultModelProvider()
);
```

## Filtering: One of the Highest-Leverage Settings

When an MCP server exposes many tools, do not pull them all in. Pick the few you actually want.

```ballerina
final ai:McpToolKit minimal = check new (
    "http://localhost:9090/mcp",
    ["searchProducts", "getProductDetails"]   // ← only these two
);
```

Why it matters:

- **Tool selection accuracy.** LLMs choose better with fewer options.
- **Prompt size.** Every tool's name, description, and parameter schema is sent on every call. Filtering trims the prompt.
- **Cost.** Smaller prompts cost less.
- **Security.** Don't expose the agent to mutation tools you didn't design for.

## Combining MCP Servers, Local Tools, and Toolkits

The agent's `tools` array accepts any combination of:

- **Function tools** marked `@ai:AgentTool`.
- **Custom toolkit classes** (your own `*ai:BaseToolKit` implementations).
- **`ai:McpToolKit` instances** for one or more remote MCP servers.

```ballerina
final ai:McpToolKit slackMcp = check new ("http://localhost:9092/mcp");
final ai:McpToolKit calendarMcp = check new ("http://localhost:9093/mcp");

final ai:Agent supportAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: "Use CRM tools for customer info, Slack for notifications, " +
            "and Calendar for scheduling follow-ups."
    },
    tools = [
        getCustomerDetails,            // local function tool
        createSupportTicket,           // local function tool
        slackMcp,                      // MCP toolkit
        calendarMcp                    // another MCP toolkit
    ],
    model = check ai:getDefaultModelProvider()
);
```

Local tools and MCP tools are completely interchangeable from the agent's perspective. The LLM sees one unified tool list.

## Connecting to Your Own MCP Service

A common pattern is for one project to **expose** a few capabilities as an MCP service ([Exposing as MCP](exposing-as-mcp.md)) and **consume** them from the agents in the same project:

```ballerina
final ai:McpToolKit internalCrm = check new ("http://localhost:9090/mcp");

final ai:Agent agent = check new (
    systemPrompt = { ... },
    tools = [internalCrm],
    model = check ai:getDefaultModelProvider()
);
```

This indirection is useful when:

- The same tools should be available to both your own agents *and* outside assistants.
- You want a clean governance boundary: auth, rate limits, and audit live on the MCP server, not in every agent that uses the tools.
- You want to swap implementations without changing every agent.

## Production Tips

- **`configurable` for endpoints.** Hard-coding MCP URLs makes environments harder to manage. Pull them from `Config.toml` instead.

  ```toml
  weatherMcpUrl = "http://localhost:9090/mcp"
  ```

  ```ballerina
  configurable string weatherMcpUrl = ?;
  final ai:McpToolKit weatherMcp = check new (weatherMcpUrl);
  ```

- **Run MCP servers close to the agent.** MCP calls are on the agent's hot path. Network latency between agent and server directly increases user-perceived response time.

- **Authenticate the connection.** If the MCP server is yours, put auth in front. If it's external, follow the vendor's auth scheme — most use OAuth or bearer tokens passed via headers.

- **Watch token bills.** Every tool the toolkit advertises is included in every reasoning step's prompt. A 50-tool MCP server, unfiltered, is a real prompt-token cost.

## What's Next

- **[Exposing a Service as MCP](exposing-as-mcp.md)** — the other side of MCP.
- **[Tools (in AI Agents)](/docs/genai/develop/agents/tools)** — back to the Add Tool dialog.
- **[Creating an Agent](/docs/genai/develop/agents/creating-an-agent)** — the agent that consumes the MCP tools.
