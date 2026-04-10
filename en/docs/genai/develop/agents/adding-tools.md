---
sidebar_position: 2
title: Adding Tools to an Agent
description: Define and register tools that let AI agents call your integration functions, APIs, and databases.
---

# Adding Tools to an Agent

Tools are Ballerina functions that an AI agent can call during its reasoning loop. They connect your agent to the real world -- APIs, databases, services, and business rules. The LLM sees each tool's description and parameter schema, then decides whether and how to call it.

In the WSO2 Integrator `ai` module, tools are plain `isolated` functions annotated with `@ai:AgentTool`. There is no special registry, no separate tool class to subclass, and no `@ai:Param` annotation -- the function signature and its doc comment are all the LLM needs.

## Defining Tools

### Basic Tool Definition

The first line of the doc comment becomes the tool description the LLM sees. Every parameter should have a `# + name - description` line so the LLM knows what to pass.

```ballerina
import ballerina/ai;
import ballerina/time;

# Returns the current date.
#
# + return - Current civil date
@ai:AgentTool
isolated function getCurrentDate() returns time:Date {
    time:Civil {year, month, day} = time:utcToCivil(time:utcNow());
    return {year, month, day};
}
```

Rules for tool functions:

- The function must be `isolated`.
- The first line of the doc comment is the tool description shown to the LLM.
- Parameter descriptions go in `# + name - description` lines.
- The return type is used to build the tool's output schema automatically.

### Tool Parameters with Descriptions

```ballerina
import ballerina/ai;

# Search for customer orders by various criteria.
#
# + customerId - Customer ID to search orders for
# + status - Order status filter: 'pending', 'shipped', 'delivered', or 'all'
# + 'limit - Maximum number of results to return (1-50)
# + return - Matching orders with status and totals
@ai:AgentTool
isolated function searchOrders(string customerId, string status = "all", int 'limit = 10)
        returns json|error {
    return check orderApi->/orders(customerId = customerId, status = status, 'limit = 'limit);
}
```

### Typed Return Values

Return a concrete record type and the agent will use its schema when presenting results to the LLM.

```ballerina
type InventoryResult record {|
    string productId;
    string productName;
    int quantityAvailable;
    string warehouseLocation;
    boolean lowStockAlert;
|};

# Check current inventory levels for a product across all warehouses.
# + productId - Product identifier
# + return - Inventory record for the product
@ai:AgentTool
isolated function checkInventory(string productId) returns InventoryResult|error {
    return check inventoryDb->queryRow(
        `SELECT product_id, product_name, quantity_available,
                warehouse_location, quantity_available < 10 AS low_stock_alert
         FROM inventory WHERE product_id = ${productId}`
    );
}
```

## Tool Categories

### Data Retrieval Tools

Read-only tools that fetch information from external systems.

```ballerina
# Run a read-only SQL query against the analytics database.
# Only SELECT queries are allowed.
#
# + query - SQL SELECT query to execute
# + return - Query result as JSON
@ai:AgentTool
isolated function queryDatabase(string query) returns json|error {
    if !query.toLowerAscii().startsWith("select") {
        return error("Only SELECT queries are allowed");
    }
    return check analyticsDb->queryRows(query);
}
```

### Action Tools

Tools that perform write operations or trigger workflows.

```ballerina
# Create a new support ticket in the ticketing system.
#
# + customerId - Customer ID who owns the ticket
# + subject - Brief subject line for the ticket
# + description - Detailed description of the issue
# + priority - Priority: 'low', 'medium', 'high', or 'critical'
# + return - The new ticket ID
@ai:AgentTool
isolated function createSupportTicket(string customerId, string subject,
        string description, string priority = "medium") returns json|error {
    return check ticketingApi->/tickets.post(
        {customerId, subject, description, priority}
    );
}
```

### Tools that Mutate Shared State

When a tool needs to update shared state, protect it with a `lock` block so it stays `isolated`.

```ballerina
import ballerina/ai;
import ballerina/time;
import ballerina/uuid;

type Task record {|
    string description;
    time:Date? dueBy;
|};

isolated map<Task> tasks = {};

# Adds a task to the to-do list.
#
# + description - What the task is about
# + dueBy - Optional due date
@ai:AgentTool
isolated function addTask(string description, time:Date? dueBy = ()) {
    lock {
        tasks[uuid:createRandomUuid()] = {description, dueBy: dueBy.clone()};
    }
}

# Lists all tasks.
# + return - All tasks currently on the to-do list
@ai:AgentTool
isolated function listTasks() returns Task[] {
    lock {
        return tasks.toArray().clone();
    }
}
```

### Connector-Based Tools

Wrap an existing WSO2 Integrator connector as an agent tool by calling it from inside the tool function.

```ballerina
import ballerina/ai;
import ballerinax/googleapis.gmail;

configurable string refreshToken = ?;
configurable string clientId = ?;
configurable string clientSecret = ?;
configurable string refreshUrl = ?;

final gmail:Client gmailClient = check new ({
    auth: {refreshToken, clientId, clientSecret, refreshUrl}
});

# Reads unread emails from the user's inbox.
# + return - List of unread Gmail messages
@ai:AgentTool
isolated function readUnreadEmails() returns gmail:Message[]|error {
    gmail:ListMessagesResponse list =
        check gmailClient->/users/me/messages(q = "label:INBOX is:unread");
    gmail:Message[]? messages = list.messages;
    if messages is () {
        return [];
    }
    return from gmail:Message m in messages
        select check gmailClient->/users/me/messages/[m.id](format = "full");
}
```

## Registering Tools with Agents

Pass the tools to the agent's `tools` field. You can pass individual functions, `ai:McpToolKit` instances, or your own `ai:BaseToolKit` classes in the same list.

### Static Registration

```ballerina
final ai:Agent supportAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: "You are a customer support assistant."
    },
    tools = [searchOrders, checkInventory, createSupportTicket],
    model = check ai:getDefaultModelProvider()
);
```

### Grouped Registration

Pre-build tool arrays when you want to share them between agents or gate certain capabilities.

```ballerina
ai:FunctionTool[] readTools = [searchOrders, checkInventory];
ai:FunctionTool[] writeTools = [createSupportTicket];

// Full-access agent
final ai:Agent fullAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: "You are a full-featured support assistant."
    },
    tools = [...readTools, ...writeTools],
    model = check ai:getDefaultModelProvider()
);

// Read-only agent
final ai:Agent readOnlyAgent = check new (
    systemPrompt = {
        role: "Support Assistant",
        instructions: "You can look up information but cannot make changes."
    },
    tools = readTools,
    model = check ai:getDefaultModelProvider()
);
```

## Toolkits

A toolkit groups related tools into a class so they can share state and configuration. Include the `ai:BaseToolKit` type into your class and return its tool configurations from `getTools()`.

```ballerina
import ballerina/ai;
import ballerina/time;
import ballerina/uuid;

public isolated class TaskManagerToolkit {
    *ai:BaseToolKit;

    private final map<Task> tasks = {};

    public isolated function getTools() returns ai:ToolConfig[] =>
        ai:getToolConfigs([self.addTask, self.listTasks]);

    # Adds a task to the to-do list.
    # + description - What the task is about
    # + dueBy - Optional due date
    @ai:AgentTool
    isolated function addTask(string description, time:Date? dueBy = ()) {
        lock {
            self.tasks[uuid:createRandomUuid()] = {description, dueBy: dueBy.clone()};
        }
    }

    # Lists all tasks.
    # + return - All tasks currently on the to-do list
    @ai:AgentTool
    isolated function listTasks() returns map<Task> {
        lock {
            return self.tasks.clone();
        }
    }
}
```

Notice the `*ai:BaseToolKit;` type inclusion -- this is how Ballerina's `ai` module composes toolkits. There is no `extends` keyword.

Register the toolkit on an agent the same way you would a function tool:

```ballerina
final ai:Agent taskAssistantAgent = check new (
    systemPrompt = {
        role: "Task Assistant",
        instructions: "You help users manage their to-do list."
    },
    tools = [new TaskManagerToolkit(), getCurrentDate],
    model = check ai:getDefaultModelProvider()
);
```

### Mixing Tools, Toolkits, and MCP

You can freely mix function tools, custom toolkits, and `ai:McpToolKit` instances in the same agent.

```ballerina
final ai:McpToolKit weatherMcp = check new ("http://localhost:9090/mcp");

final ai:Agent travelAgent = check new (
    systemPrompt = {
        role: "Travel Assistant",
        instructions: "You help users plan trips end-to-end."
    },
    tools = [searchHotels, weatherMcp, new TaskManagerToolkit()],
    model = check ai:getDefaultModelProvider()
);
```

## Tool Design Best Practices

1. **Clear descriptions** -- The first line of your doc comment is the tool description. Be specific about what the tool does and when to use it.
2. **Describe every parameter** -- Add a `# + name - description` line for each argument.
3. **Informative errors** -- Return descriptive error messages so the LLM can reason about failures.
4. **Limited output** -- Trim large responses to prevent exceeding context window limits.

### Informative Error Example

```ballerina
# Look up a customer by ID. If not found, suggests alternative search methods.
# + customerId - Customer identifier
# + return - Customer record or a structured not-found response
@ai:AgentTool
isolated function getCustomer(string customerId) returns json|error {
    json|error result = crmClient->/customers/[customerId];
    if result is error {
        return {
            "found": false,
            "message": string `No customer found with ID '${customerId}'.`,
            "suggestion": "Try searchCustomerByEmail to search by email address instead."
        };
    }
    return result;
}
```

### Confirmation-Required Tools

For sensitive actions, bake the confirmation requirement into the tool description so the agent knows to check with the user first.

```ballerina
# Cancel a customer order. IMPORTANT: Always confirm with the customer before
# calling this tool. This action cannot be undone.
#
# + orderId - Order to cancel
# + reason - Reason provided by the customer
# + return - Cancellation confirmation payload
@ai:AgentTool
isolated function cancelOrder(string orderId, string reason) returns json|error {
    return check orderApi->/orders/[orderId]/cancel.post({reason});
}
```

## What's Next

- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build your first agent
- [Adding Memory](/docs/genai/develop/agents/adding-memory) -- Configure conversation history
- [Advanced Configuration](/docs/genai/develop/agents/advanced-config) -- Multi-agent orchestration
