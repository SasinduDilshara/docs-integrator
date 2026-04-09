---
title: Building a Customer Care Agent with MCP
---

# Building a Customer Care Agent with MCP

This example demonstrates how to build an MCP (Model Context Protocol) server that exposes your enterprise database and APIs as tools that any MCP-compatible AI assistant can discover and call. This approach turns your existing data sources into a universal tool layer -- connect once in WSO2 Integrator, and every LLM client in your organization can query customers, look up orders, and get product information through a standardized protocol.

## What You'll Build

An MCP server with three tools:

- **queryCustomers** -- Search and retrieve customer records from a database
- **lookupOrders** -- Look up order details including status, items, and shipping info
- **getProductInfo** -- Retrieve product specifications, pricing, and availability

The server runs as an MCP-compatible endpoint that any AI assistant (Claude Desktop, VS Code Copilot, custom agents) can connect to.

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- A running database (this example uses MySQL; any supported database works)
- Familiarity with [What is MCP?](../key-concepts/what-is-mcp/index.md)

:::info
This example connects to a MySQL database. WSO2 Integrator supports all major databases -- see [Database Connectors](/docs/connectors/databases) for connection configuration.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `customer-care-mcp` and choose a directory.

## Step 2: Connect to the Database

Set up the database connection that your MCP tools will use:

```ballerina
import ballerinax/mysql;
import ballerina/sql;

configurable string dbHost = "localhost";
configurable int dbPort = 3306;
configurable string dbUser = ?;
configurable string dbPassword = ?;
configurable string dbName = "enterprise";

final mysql:Client dbClient = check new (
    host = dbHost,
    port = dbPort,
    user = dbUser,
    password = dbPassword,
    database = dbName
);
```

## Step 3: Define the MCP Tools

Create three tools that expose your enterprise data. Each tool includes a description that helps LLMs understand when and how to use it.

### Customer Query Tool

```ballerina
import ballerinax/ai.mcp;

type Customer record {|
    int id;
    string name;
    string email;
    string company;
    string tier; // "free", "pro", "enterprise"
    string createdAt;
|};

@mcp:Tool {
    description: "Search for customers by name, email, or company. Returns matching customer records with their account tier and creation date. Use this when the user asks about a specific customer or wants to look up account information."
}
function queryCustomers(string searchTerm, int 'limit = 10) returns Customer[]|error {
    sql:ParameterizedQuery query = `SELECT id, name, email, company, tier, created_at as createdAt
        FROM customers
        WHERE name LIKE ${"%" + searchTerm + "%"}
           OR email LIKE ${"%" + searchTerm + "%"}
           OR company LIKE ${"%" + searchTerm + "%"}
        LIMIT ${'limit}`;

    stream<Customer, sql:Error?> resultStream = dbClient->query(query);
    return from Customer customer in resultStream select customer;
}
```

### Order Lookup Tool

```ballerina
type OrderDetail record {|
    int orderId;
    string customerName;
    string status;
    decimal totalAmount;
    string currency;
    string createdAt;
    string? shippedAt;
    string? trackingNumber;
|};

type OrderItem record {|
    string productName;
    int quantity;
    decimal unitPrice;
|};

type OrderResponse record {|
    OrderDetail 'order;
    OrderItem[] items;
|};

@mcp:Tool {
    description: "Look up a specific order by order ID. Returns order status, total amount, shipping information, and line items. Use this when the user asks about order details, delivery status, or order contents."
}
function lookupOrder(int orderId) returns OrderResponse|error {
    OrderDetail orderDetail = check dbClient->queryRow(
        `SELECT o.id as orderId, c.name as customerName, o.status, o.total_amount as totalAmount,
                o.currency, o.created_at as createdAt, o.shipped_at as shippedAt, o.tracking_number as trackingNumber
         FROM orders o JOIN customers c ON o.customer_id = c.id
         WHERE o.id = ${orderId}`
    );

    stream<OrderItem, sql:Error?> itemStream = dbClient->query(
        `SELECT p.name as productName, oi.quantity, oi.unit_price as unitPrice
         FROM order_items oi JOIN products p ON oi.product_id = p.id
         WHERE oi.order_id = ${orderId}`
    );

    OrderItem[] items = check from OrderItem item in itemStream select item;

    return {'order: orderDetail, items};
}
```

### Product Info Tool

```ballerina
type Product record {|
    int id;
    string name;
    string category;
    string description;
    decimal price;
    string currency;
    int stockQuantity;
    boolean isActive;
|};

@mcp:Tool {
    description: "Get product information by name or category. Returns product details including pricing, stock availability, and description. Use this when the user asks about product specifications, pricing, or availability."
}
function getProductInfo(string searchTerm, string? category = ()) returns Product[]|error {
    sql:ParameterizedQuery query;
    if category is string {
        query = `SELECT id, name, category, description, price, currency, stock_quantity as stockQuantity, is_active as isActive
            FROM products
            WHERE (name LIKE ${"%" + searchTerm + "%"} OR description LIKE ${"%" + searchTerm + "%"})
              AND category = ${category} AND is_active = true`;
    } else {
        query = `SELECT id, name, category, description, price, currency, stock_quantity as stockQuantity, is_active as isActive
            FROM products
            WHERE (name LIKE ${"%" + searchTerm + "%"} OR description LIKE ${"%" + searchTerm + "%"})
              AND is_active = true`;
    }

    stream<Product, sql:Error?> resultStream = dbClient->query(query);
    return from Product product in resultStream select product;
}
```

## Step 4: Configure the MCP Server

Register all tools with the MCP server and configure the endpoint:

```ballerina
final mcp:Server mcpServer = check new ({
    name: "Enterprise Data MCP Server",
    version: "1.0.0",
    description: "Provides access to customer, order, and product data from the enterprise database."
});

public function init() returns error? {
    check mcpServer.registerTool(queryCustomers);
    check mcpServer.registerTool(lookupOrder);
    check mcpServer.registerTool(getProductInfo);
}
```

## Step 5: Add Security

Protect your MCP server with authentication and rate limiting:

```ballerina
configurable string mcpApiKey = ?;

// The MCP server configuration supports built-in API key validation
final mcp:Server mcpServer = check new ({
    name: "Enterprise Data MCP Server",
    version: "1.0.0",
    description: "Provides access to customer, order, and product data.",
    auth: {
        apiKey: mcpApiKey
    }
});
```

:::warning
Always secure MCP servers that expose enterprise data. Use API keys at minimum, and consider OAuth2 for multi-tenant deployments. See [Building AI Agents with MCP Servers](../build-ai-applications/mcp-integration/building-agents-with-mcp-servers.md) for authentication patterns.
:::

## Step 6: Configure and Run

Add your credentials to `Config.toml`:

```toml
dbHost = "localhost"
dbPort = 3306
dbUser = "app_user"
dbPassword = "<YOUR_DB_PASSWORD>"
dbName = "enterprise"
mcpApiKey = "<YOUR_MCP_API_KEY>"
```

:::warning
Never commit database credentials or API keys to version control. Use environment variables or a secrets manager in production. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption).
:::

## Test with MCP Inspector

1. Click **Run** in VS Code.
2. Open the MCP Inspector from the toolbar.
3. Browse the available tools and their schemas.
4. Call each tool interactively:

```
Tool: queryCustomers
Input: {"searchTerm": "acme", "limit": 5}
Result: [{"id": 1, "name": "Acme Corp", "email": "admin@acme.com", ...}]

Tool: lookupOrder
Input: {"orderId": 1042}
Result: {"order": {"orderId": 1042, "status": "shipped", ...}, "items": [...]}
```

## Connect from an External AI Assistant

Add this configuration to your MCP-compatible client (e.g., Claude Desktop):

```json
{
  "mcpServers": {
    "enterprise-data": {
      "url": "http://localhost:9090/mcp",
      "apiKey": "<YOUR_MCP_API_KEY>"
    }
  }
}
```

Now you can ask natural language questions in your AI assistant:

```
You: How many orders does Acme Corp have?
Assistant: [Calls queryCustomers, then lookupOrder] Acme Corp has 12 orders...

You: What's the status of order 1042?
Assistant: [Calls lookupOrder] Order 1042 shipped on March 5 with tracking number TRK-78901...
```

:::tip
The MCP Inspector shows the exact tool calls, inputs, and outputs -- use it to verify your tools behave correctly before connecting external clients.
:::

## Extend It

- **Add write tools** -- Create tools for updating customer records or creating new orders with proper authorization checks.
- **Connect additional data sources** -- Add tools that query REST APIs, file storage, or message queues alongside the database.
- **Add audit logging** -- Log every tool invocation for compliance. See [AI Agent Observability](../build-ai-applications/ai-agents/ai-agent-observability.md).

## Source Code

Find the complete working project on GitHub: [mcp-enterprise-data example](https://github.com/wso2/integrator-examples/tree/main/genai/mcp-enterprise-data)

## What's Next

- [Creating an MCP Server](../build-ai-applications/mcp-integration/exposing-services-as-mcp-server.md) -- Deep dive into MCP server configuration
- [Building AI Agents with MCP Servers](../build-ai-applications/mcp-integration/building-agents-with-mcp-servers.md) -- Connect agents to external MCP servers
- [AI Governance and Security](../reference/ai-governance-and-security.md) -- Production security patterns
