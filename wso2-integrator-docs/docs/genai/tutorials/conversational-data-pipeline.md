---
sidebar_position: 5
title: "Build a Conversational Data Pipeline"
description: Create an agent that queries databases, transforms data, and generates reports through natural language in WSO2 Integrator.
---

# Build a Conversational Data Pipeline

**Time:** 30-45 minutes

You will build an agent that lets users query databases, transform data, and generate reports using natural language. Instead of writing SQL or building dashboards, users simply ask questions like "Show me top customers by revenue this quarter" and the agent handles querying, aggregation, formatting, and presentation. This tutorial combines agent tool binding with data transformation and report generation into a practical, self-service analytics integration.

## What You'll Build

A conversational analytics agent with four capabilities:

- **Database query tools** — Execute parameterized queries against multiple data sources
- **Data transformation tools** — Aggregate, filter, sort, and pivot result sets
- **Report generation** — Format results as tables, summaries, or CSV exports
- **Guardrails** — Prevent destructive queries and limit result set sizes

## What You'll Learn

- How to expose database queries as agent tools safely
- How to build data transformation tools for common analytics operations
- How to generate formatted reports from agent responses
- How to add guardrails to prevent misuse

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- An OpenAI API key
- A running database with sample data (this tutorial uses MySQL)

:::info
This tutorial uses MySQL. WSO2 Integrator supports all major databases — see [Database Connectors](/docs/connectors/databases) for connection configuration.
:::

## Step 1: Create a New Project

1. Open VS Code and press **Ctrl+Shift+P** (or **Cmd+Shift+P** on macOS).
2. Select **WSO2 Integrator: Create New Project**.
3. Name the project `conversational-data-pipeline` and choose a directory.

## Step 2: Connect to Data Sources

Set up database connections for your analytics data:

```ballerina
import ballerinax/mysql;
import ballerina/sql;
import ballerinax/ai.agent;
import ballerinax/openai.chat as openai;

configurable string openaiApiKey = ?;
configurable string dbHost = "localhost";
configurable int dbPort = 3306;
configurable string dbUser = ?;
configurable string dbPassword = ?;
configurable string dbName = "analytics";

final mysql:Client dbClient = check new (
    host = dbHost,
    port = dbPort,
    user = dbUser,
    password = dbPassword,
    database = dbName
);
```

## Step 3: Define Query Tools

Create tools that let the agent query specific data domains. Each tool is scoped to a particular table or view to maintain security boundaries.

```ballerina
type SalesRecord record {|
    string customerName;
    string product;
    decimal amount;
    string currency;
    string saleDate;
    string region;
|};

@agent:Tool {
    description: "Query sales data with optional filters. Supports filtering by date range, region, and product category. Returns individual sales records. Use this for questions about sales transactions, revenue, or customer purchases."
}
isolated function querySales(string? startDate = (), string? endDate = (), string? region = (), string? product = (), int 'limit = 100) returns SalesRecord[]|error {
    // Cap the limit to prevent excessive data retrieval
    int effectiveLimit = 'limit > 500 ? 500 : 'limit;

    sql:ParameterizedQuery query = `SELECT c.name as customerName, p.name as product,
            s.amount, s.currency, s.sale_date as saleDate, s.region
        FROM sales s
        JOIN customers c ON s.customer_id = c.id
        JOIN products p ON s.product_id = p.id
        WHERE 1=1`;

    if startDate is string {
        query = sql:queryConcat(query, ` AND s.sale_date >= ${startDate}`);
    }
    if endDate is string {
        query = sql:queryConcat(query, ` AND s.sale_date <= ${endDate}`);
    }
    if region is string {
        query = sql:queryConcat(query, ` AND s.region = ${region}`);
    }
    if product is string {
        query = sql:queryConcat(query, ` AND p.name LIKE ${"%" + product + "%"}`);
    }

    query = sql:queryConcat(query, ` ORDER BY s.sale_date DESC LIMIT ${effectiveLimit}`);

    stream<SalesRecord, sql:Error?> resultStream = dbClient->query(query);
    return from SalesRecord rec in resultStream select rec;
}

type CustomerSummary record {|
    string customerName;
    string company;
    decimal totalRevenue;
    int orderCount;
    string lastOrderDate;
    string tier;
|};

@agent:Tool {
    description: "Get customer summaries with aggregated metrics. Returns total revenue, order count, and last order date per customer. Use this for questions about top customers, customer segments, or account health."
}
isolated function getCustomerSummaries(string? tier = (), string? sortBy = "totalRevenue", int 'limit = 20) returns CustomerSummary[]|error {
    int effectiveLimit = 'limit > 100 ? 100 : 'limit;
    string orderColumn = sortBy == "orderCount" ? "order_count" : "total_revenue";

    sql:ParameterizedQuery query = `SELECT c.name as customerName, c.company,
            SUM(s.amount) as totalRevenue, COUNT(s.id) as orderCount,
            MAX(s.sale_date) as lastOrderDate, c.tier
        FROM customers c
        LEFT JOIN sales s ON c.id = s.customer_id`;

    if tier is string {
        query = sql:queryConcat(query, ` WHERE c.tier = ${tier}`);
    }

    query = sql:queryConcat(query, ` GROUP BY c.id, c.name, c.company, c.tier`);
    query = sql:queryConcat(query, ` ORDER BY ${orderColumn} DESC LIMIT ${effectiveLimit}`);

    stream<CustomerSummary, sql:Error?> resultStream = dbClient->query(query);
    return from CustomerSummary rec in resultStream select rec;
}
```

:::warning
Never allow agents to execute raw SQL. Always use parameterized queries and scope each tool to specific tables with read-only access. This prevents SQL injection and limits the blast radius of unexpected queries.
:::

## Step 4: Add Data Transformation Tools

Create tools for common analytics operations — aggregation, comparison, and trend analysis:

```ballerina
type RegionSummary record {|
    string region;
    decimal totalRevenue;
    int transactionCount;
    decimal averageOrderValue;
|};

@agent:Tool {
    description: "Get revenue breakdown by region for a given time period. Returns total revenue, transaction count, and average order value per region. Use this for geographic performance analysis."
}
isolated function getRevenueByRegion(string startDate, string endDate) returns RegionSummary[]|error {
    sql:ParameterizedQuery query = `SELECT region, SUM(amount) as totalRevenue,
            COUNT(*) as transactionCount, AVG(amount) as averageOrderValue
        FROM sales
        WHERE sale_date BETWEEN ${startDate} AND ${endDate}
        GROUP BY region
        ORDER BY totalRevenue DESC`;

    stream<RegionSummary, sql:Error?> resultStream = dbClient->query(query);
    return from RegionSummary rec in resultStream select rec;
}

type TrendData record {|
    string period;
    decimal revenue;
    int orderCount;
|};

@agent:Tool {
    description: "Get revenue and order trends over time, grouped by month. Use this for time-series analysis, growth tracking, or seasonal pattern detection."
}
isolated function getRevenueTrend(string startDate, string endDate) returns TrendData[]|error {
    sql:ParameterizedQuery query = `SELECT DATE_FORMAT(sale_date, '%Y-%m') as period,
            SUM(amount) as revenue, COUNT(*) as orderCount
        FROM sales
        WHERE sale_date BETWEEN ${startDate} AND ${endDate}
        GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
        ORDER BY period ASC`;

    stream<TrendData, sql:Error?> resultStream = dbClient->query(query);
    return from TrendData rec in resultStream select rec;
}
```

## Step 5: Build the Report Generation Tool

Add a tool that formats query results into structured reports:

```ballerina
import ballerina/io;

@agent:Tool {
    description: "Generate a formatted report as a CSV string from provided data. Use this when the user asks to export or download data."
}
isolated function generateCsvReport(string title, string[] headers, string[][] rows) returns string|error {
    string csvContent = title + "\n\n";
    csvContent += string:'join(",", ...headers) + "\n";

    foreach string[] row in rows {
        csvContent += string:'join(",", ...row) + "\n";
    }

    // In production, write to file storage or return a download link
    string fileName = string `/tmp/${title.replaceAll(" ", "_")}.csv`;
    check io:fileWriteString(fileName, csvContent);

    return string `Report generated: ${fileName} (${rows.length()} rows)`;
}
```

## Step 6: Configure the Agent

Wire everything together with a system prompt that guides the agent's analytics behavior:

```ballerina
final agent:ChatAgent dataAgent = check new (
    systemPrompt = string `You are a data analytics assistant. You help users explore business data through natural language.

        Capabilities:
        - Query sales data by date range, region, or product
        - Retrieve customer summaries and segment analysis
        - Analyze revenue by region and over time
        - Generate CSV reports for export

        Guidelines:
        - When the user asks a vague question, ask clarifying questions about time range, filters, or scope.
        - Present data in clean, readable tables using markdown formatting.
        - When showing numbers, include currency symbols and format large numbers with commas.
        - Proactively suggest follow-up analyses (e.g., "Would you like to see this broken down by region?").
        - For trend questions, default to the last 12 months if no date range is specified.
        - If the data set is large, summarize the key insights and offer to export the full data.`,
    model = check new openai:Client({auth: {token: openaiApiKey}}),
    memory = new agent:MessageWindowChatMemory(maxMessages = 30),
    tools = [querySales, getCustomerSummaries, getRevenueByRegion, getRevenueTrend, generateCsvReport]
);
```

## Step 7: Add Guardrails

Prevent misuse by adding input validation and output limits:

```ballerina
import ballerina/time;

// Validate date ranges to prevent scanning the entire database
function validateDateRange(string startDate, string endDate) returns error? {
    time:Utc startUtc = check time:utcFromString(startDate + "T00:00:00Z");
    time:Utc endUtc = check time:utcFromString(endDate + "T00:00:00Z");

    // Maximum 2-year range
    time:Seconds diff = time:utcDiffSeconds(endUtc, startUtc);
    if diff > 63072000 {
        return error("Date range cannot exceed 2 years. Please narrow your query.");
    }
}
```

:::tip
For production deployments, add token budgets and rate limiting to prevent runaway costs. See [Token & Cost Management](/docs/genai/guardrails/token-cost-management).
:::

## Step 8: Configure and Run

Add your settings to `Config.toml`:

```toml
openaiApiKey = "<YOUR_OPENAI_API_KEY>"
dbHost = "localhost"
dbPort = 3306
dbUser = "analytics_reader"
dbPassword = "<YOUR_DB_PASSWORD>"
dbName = "analytics"
```

:::warning
Use a read-only database user for the analytics agent. Never grant write permissions to tools that are exposed to natural language queries. See [Secrets & Encryption](/docs/deploy-operate/secure/secrets-encryption).
:::

## Test It

1. Click **Run** in VS Code.
2. Open the built-in chat interface.
3. Try natural language analytics queries:

```
You: Who are our top 5 customers by revenue?
Agent: Here are your top 5 customers by total revenue:

| Customer       | Company      | Total Revenue | Orders | Tier       |
|----------------|--------------|---------------|--------|------------|
| Acme Corp      | Acme Inc     | $234,500      | 42     | Enterprise |
| GlobalTech     | GT Holdings  | $189,200      | 38     | Enterprise |
| StartupXYZ     | XYZ Labs     | $78,400       | 15     | Pro        |
| DataFlow Inc   | DataFlow     | $62,100       | 22     | Pro        |
| CloudFirst     | CF Group     | $45,800       | 11     | Pro        |

Would you like to see this broken down by region or time period?

You: Show me the revenue trend for the last 6 months
Agent: Here's your monthly revenue trend from September 2025 to February 2026:

| Month    | Revenue    | Orders | vs Previous |
|----------|------------|--------|-------------|
| Sep 2025 | $87,300    | 156    | —           |
| Oct 2025 | $92,100    | 168    | +5.5%       |
| Nov 2025 | $110,400   | 201    | +19.9%      |
| Dec 2025 | $125,600   | 234    | +13.8%      |
| Jan 2026 | $98,200    | 178    | -21.8%      |
| Feb 2026 | $105,800   | 189    | +7.7%       |

Revenue peaked in December (likely seasonal) with a post-holiday dip in January.
Would you like me to export this data as a CSV?

You: Yes, export it
Agent: Report generated: /tmp/Revenue_Trend.csv (6 rows). The file is ready for download.
```

:::tip
Open the **Trace** panel to see which tools the agent calls, the SQL parameters it generates, and how it transforms the results into readable tables.
:::

## Extend It

- **Add chart generation** — Integrate a charting library to produce visual charts from query results.
- **Connect multiple databases** — Add tools for different databases (e.g., product analytics, financial data) and let the agent query across sources.
- **Add scheduled reports** — Combine the agent with an automation to generate and email weekly reports automatically.

## Source Code

Find the complete working project on GitHub: [conversational-data-pipeline example](https://github.com/wso2/integrator-examples/tree/main/genai/conversational-data-pipeline)

## What's Next

- [Tool Binding (Function Calling)](/docs/genai/agents/tool-binding) — Advanced patterns for connecting agents to data sources
- [Input / Output Guardrails](/docs/genai/guardrails/input-output-guardrails) — Protect agents from prompt injection and data leakage
- [Deploy to Production](/docs/deploy-operate/deploy/docker-kubernetes) — Containerize and deploy your data pipeline agent
