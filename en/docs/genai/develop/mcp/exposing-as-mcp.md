---
sidebar_position: 2
title: Exposing a Service as MCP
description: Reference for the MCP Service artifact in WSO2 Integrator — listeners, the Tools panel, and per-tool configuration.
---

# Exposing a Service as MCP

An **MCP Service** is a WSO2 Integrator artifact that publishes a set of tools over the Model Context Protocol. Any MCP-compatible client — Claude Desktop, GitHub Copilot, or another AI agent — can connect, discover the tools, and call them.

## Creating an MCP Service

1. Click **+ Add Artifact** on the project view.
2. In the **Artifacts** page, under the **AI Integration** section pick **MCP Service** — sitting next to AI Chat Agent:

   ![Artifacts page showing the AI Integration category with two artifacts: AI Chat Agent (highlighted) and MCP Service. Above is Automation; below is Integration as API (HTTP Service, GraphQL Service Beta, TCP Service Beta).](/img/genai/develop/shared/07-artifacts-page-full.png)

3. The MCP Service editor opens.

## The MCP Service Editor

![The MCP Service editor. Top: 'MCP Service - Implement and configure your service' with Configure and Try It buttons in the top-right. A 'Listener: mcpListener' chip appears below the title. The Tools section is empty with the message 'No tools found. Add a new tool.' and a + Add Tool button.](/img/genai/develop/mcp/01-mcp-service-overview.png)

The editor surfaces three things at the top level:

| Element | What it does |
|---|---|
| **Listener** | The `mcp:Listener` the service runs on. Click to swap it for a different listener (different port, different transport). |
| **Tools** | The tools the service exposes. Empty by default — click **+ Add Tool** to start. |
| **Configure** | Service-level settings — base path, advanced listener tuning. |
| **Try It** | Send sample MCP requests to the service from inside the editor. |

## The Listener

Every MCP Service runs on an `mcp:Listener`. When you create the service, BI generates a default `mcpListener` connection on a port (usually 9090) with the **Streamable HTTP** transport — the modern HTTP-based variant that supports streaming responses.

| Transport | Use when… |
|---|---|
| **Streamable HTTP** (default) | Remote clients, web-based AI assistants, anything that talks HTTP. The default and recommended transport. |
| **stdio** | Local clients like Claude Desktop, where the assistant launches your service as a subprocess and pipes JSON over stdin/stdout. |

You can edit the listener from the Listener chip on the editor or from the **Listeners** category in the left sidebar.

## Adding a Tool

Click **+ Add Tool** to open the **Tool Configuration** panel:

![The Tool Configuration panel for an MCP Service. Fields visible: Tool Name (with 'missing identifier' warning), Tool Description, Parameters with + Add Parameter, Return Type. Cancel and Save buttons at the bottom.](/img/genai/develop/mcp/02-mcp-tool-configuration.png)

| Field | Required | What it does |
|---|---|---|
| **Tool Name** | Yes | The name AI clients will see. Camel-case, descriptive: `getOrderStatus`, `searchProducts`. |
| **Tool Description** | Yes | What the tool does **and when** the assistant should use it. The single most important field for tool discoverability. |
| **Parameters** | No | Each parameter has a name, type, and description. The description is included in the schema sent to the client. |
| **Return Type** | Yes | The Ballerina type returned. The schema is generated from this and shown to the client. |

A typical filled-in tool form, with two `int` parameters and the Return Type picker open:

![The Tool Configuration panel filled in. Tool Name* set to 'add'. Tool Description set to 'Add two numbers'. Parameters section showing two rows: 'int / a' (with edit/delete icons) and 'int / b' (with edit/delete icons), plus + Add Parameter link. Return Type* with type picker dropdown OPEN showing search box, Primitive Types section (string, int, float, decimal, boolean), and at the bottom + Create New Type and Open Type Browser links.](/img/genai/develop/mcp/04-mcp-tool-return-type-picker.png)

The Return Type picker shows:

| Section | What's in it |
|---|---|
| **Search** | Filter all visible types by name. |
| **Primitive Types** | `string`, `int`, `float`, `decimal`, `boolean`, `()` (nil). |
| **+ Create New Type** | Open the Type editor to define a record on the spot — useful when the return is structured. |
| **Open Type Browser** | Browse every type defined in the project plus the imported standard library. |

After **Save**, BI generates a `remote function` on the MCP service:

```ballerina
import ballerina/mcp;

listener mcp:Listener mcpListener = new (9090);

service mcp:Service /mcp on mcpListener {

    # Get the current status of a customer order by order ID.
    #
    # + orderId - Customer order identifier (format ORD-XXXXX)
    # + return - Current status, ETA, and tracking number
    remote function getOrderStatus(string orderId) returns OrderStatus|error {
        return check orderApi->/orders/[orderId]/status;
    }
}
```

The function name → tool name. The doc comment → tool description. The parameter doc lines → parameter descriptions. The return type → output schema. Nothing else to maintain.

## Writing a Good Tool Description

For each tool you add, the description is what the AI client reads to decide *whether and when* to use it. Two patterns matter most:

### Be explicit about scope

> *"Search the product catalog by keyword. Returns up to 10 matching products with name, price, and availability. Use for general product discovery questions."*

> *"Retrieves a single product by its exact SKU. Use only when the user provides a SKU."*

These two are clearly different and the AI will not confuse them.

### Bake in safety rules

> *"Cancel a customer order. **IMPORTANT: Always confirm with the customer before calling this tool.** This action cannot be undone."*

The AI client will follow it most of the time. (For deterministic enforcement, also keep server-side checks.)

### Don't leave it generic

> *"Customer endpoint."* ❌

A generic description leads to bad tool selection downstream. If you can't write a useful one-liner, the tool is probably too vague to be useful.

## Advanced: `mcp:AdvancedService`

When tool definitions need to come from configuration, a database, or some runtime source — not from compile-time `remote function`s — use `mcp:AdvancedService`. You implement two callbacks:

- **`onListTools()`** — return the list of tools you advertise.
- **`onCallTool(params, session)`** — dispatch a call by name to the right handler.

```ballerina
service mcp:AdvancedService /mcp on mcpListener {

    isolated remote function onListTools()
            returns mcp:ListToolsResult|mcp:ServerError => {
        tools: [
            {
                name: "getCurrentWeather",
                description: "Get current weather conditions for a location",
                inputSchema: {
                    "type": "object",
                    "properties": {
                        "city": {"type": "string", "description": "City name"}
                    },
                    "required": ["city"]
                }
            }
        ]
    };

    isolated remote function onCallTool(mcp:CallToolParams params, mcp:Session? session)
            returns mcp:CallToolResult|mcp:ServerError {
        if params.name == "getCurrentWeather" {
            record {|string city;|} args = check params.arguments.cloneWithType();
            Weather w = check getCurrentWeather(args.city);
            return {content: [{'type: "text", text: w.toJsonString()}]};
        }
        return error("Unknown tool: " + params.name);
    }
}
```

| Use `mcp:Service` when… | Use `mcp:AdvancedService` when… |
|---|---|
| Tools map cleanly to compile-time `remote function`s. | Tools are dynamic — defined in configuration, a database, or another service. |
| You want BI to derive schemas automatically. | You want custom input schemas or extra validation. |
| You want the simplest possible MCP server. | You want a dispatch-table architecture. |

> Note: in `onCallTool`, content items use the field name `'type` (with a leading quote) because `type` is a reserved keyword in Ballerina.

## Configuring for an MCP Client

Once your service is running, MCP clients connect by URL.

**Claude Desktop**:

```json
{
  "mcpServers": {
    "wso2-integrator": {
      "url": "http://localhost:9090/mcp"
    }
  }
}
```

**Another agent inside WSO2 Integrator**: see [Consuming MCP from an Agent](consuming-mcp-from-agent.md).

## Error Handling

Tools should return informative errors so AI clients can recover or suggest alternatives:

```ballerina
remote function getInvoice(string invoiceId) returns json|error {
    if !invoiceId.startsWith("INV-") {
        return error(string `Invalid invoice ID '${invoiceId}'. Expected format INV-XXXXX.`);
    }
    return check billingApi->/invoices/[invoiceId];
}
```

Avoid bare `error()` with no message. The AI client will pass the error message back into its reasoning loop, so a good message helps it suggest the right next step to the user.

## Operational Notes

- **Keep the tool list small.** Fewer, better-described tools beat dozens of overlapping ones.
- **Authenticate.** Anything you expose over MCP is, by default, accessible to whoever connects. Add auth (HTTP basic, OAuth, custom headers) on the listener.
- **Log tool calls.** Just like any other API surface — knowing who called what is essential for debugging and audit.
- **Version explicitly.** When tool shapes change, prefer adding a new tool over silently changing an existing one. AI clients depend on stable schemas.

## What's Next

- **[Consuming MCP from an Agent](consuming-mcp-from-agent.md)** — the other half of the MCP picture.
- **[Tools (in AI Agents)](/docs/genai/develop/agents/tools)** — local-tool reference; the same description-quality rules apply.
