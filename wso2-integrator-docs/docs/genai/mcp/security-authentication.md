---
sidebar_position: 4
title: "Secure MCP Endpoints"
description: "Add authentication, rate limiting, and audit logging to your MCP servers and clients."
---

# Secure MCP Endpoints

MCP servers expose your integrations as tools that any LLM can call. Without proper security, this creates a direct path from external AI applications to your internal services. You need authentication to control who can connect, rate limiting to prevent abuse, and audit logging to track every tool invocation.

This guide covers securing both MCP servers you expose and MCP clients that connect to external servers.

## Authentication

### Bearer Token Authentication

The simplest approach for server-to-server MCP connections. The client sends a token in the authorization header, and your MCP server validates it before executing any tool.

```ballerina
import ballerinax/ai.mcp;
import ballerina/http;

configurable string[] allowedTokens = ?;

@mcp:ServiceConfig {
    name: "secure-weather-service",
    version: "1.0.0",
    description: "Weather tools with authentication",
    auth: {
        type: "bearer",
        validator: validateToken
    }
}
service on new mcp:Listener(3000) {

    @mcp:Tool {
        name: "getCurrentWeather",
        description: "Get current weather for a city",
        parameters: {
            location: {type: "string", description: "City name", required: true}
        }
    }
    resource function post getCurrentWeather(string location) returns WeatherData|error {
        return getWeather(location, "celsius");
    }
}

function validateToken(string token) returns boolean {
    return allowedTokens.indexOf(token) is int;
}
```

### OAuth 2.0 Authentication

For production deployments where you need fine-grained access control, use OAuth 2.0 with scopes:

```ballerina
@mcp:ServiceConfig {
    name: "enterprise-tools",
    version: "1.0.0",
    description: "Enterprise integration tools",
    auth: {
        type: "oauth2",
        introspectionUrl: "https://identity.example.com/oauth2/introspect",
        requiredScopes: ["mcp:read", "mcp:execute"]
    }
}
service on new mcp:Listener(3000) {
    // Tools are only accessible with valid OAuth tokens that have the required scopes
}
```

:::info
For detailed OAuth 2.0 configuration, including identity provider setup and token management, see [Authentication](/docs/deploy-operate/secure/authentication).
:::

### Authenticating as an MCP Client

When your integration connects to an external MCP server that requires authentication, provide credentials in the client configuration:

```ballerina
import ballerinax/ai.mcp;

configurable string mcpBearerToken = ?;

final mcp:Client secureMcpClient = check new ({
    url: "https://tools.example.com/mcp",
    transport: "sse",
    auth: {
        token: mcpBearerToken
    }
});
```

For OAuth 2.0 client credentials:

```ballerina
final mcp:Client secureMcpClient = check new ({
    url: "https://tools.example.com/mcp",
    transport: "sse",
    auth: {
        type: "oauth2",
        tokenUrl: "https://identity.example.com/oauth2/token",
        clientId: clientId,
        clientSecret: clientSecret,
        scopes: ["mcp:read", "mcp:execute"]
    }
});
```

## Rate Limiting

Prevent abuse by limiting how many tool calls a client can make within a given time window:

```ballerina
@mcp:ServiceConfig {
    name: "rate-limited-tools",
    version: "1.0.0",
    description: "Tools with rate limiting",
    rateLimit: {
        maxRequests: 100,
        windowSeconds: 60,       // 100 requests per minute
        strategy: "sliding"      // sliding window or fixed window
    }
}
service on new mcp:Listener(3000) {
    // Clients exceeding the rate limit receive a 429 Too Many Requests response
}
```

### Per-Tool Rate Limits

Apply different limits to different tools based on their cost or sensitivity:

```ballerina
@mcp:Tool {
    name: "searchDatabase",
    description: "Search the product database",
    parameters: {
        query: {type: "string", description: "Search query", required: true}
    },
    rateLimit: {
        maxRequests: 30,
        windowSeconds: 60       // More restrictive — database queries are expensive
    }
}
resource function post searchDatabase(string query) returns SearchResult[]|error {
    return performSearch(query);
}

@mcp:Tool {
    name: "getServerTime",
    description: "Get the current server time",
    rateLimit: {
        maxRequests: 300,
        windowSeconds: 60       // More permissive — lightweight operation
    }
}
resource function post getServerTime() returns string {
    return time:utcNow().toString();
}
```

:::tip
Set rate limits based on the actual cost of each tool. Database queries, external API calls, and write operations should have tighter limits than simple lookups or read-only operations.
:::

## Audit Logging

Track every tool invocation for compliance, debugging, and usage analysis:

```ballerina
import ballerina/log;
import ballerina/time;

@mcp:ServiceConfig {
    name: "audited-tools",
    version: "1.0.0",
    description: "Tools with audit logging",
    interceptors: [new AuditInterceptor()]
}
service on new mcp:Listener(3000) {
    // All tool calls pass through the audit interceptor
}

service class AuditInterceptor {
    *mcp:Interceptor;

    remote function onToolCall(mcp:ToolCallContext ctx, mcp:ToolRequest request) returns json|error {
        time:Utc startTime = time:utcNow();
        string clientId = ctx.clientId ?: "anonymous";

        log:printInfo("MCP tool invocation started",
            tool = request.toolName,
            clientId = clientId,
            parameters = request.parameters.toString()
        );

        // Execute the actual tool
        json|error result = ctx.next(request);

        time:Utc endTime = time:utcNow();
        decimal duration = <decimal>(endTime[0] - startTime[0]);

        if result is error {
            log:printError("MCP tool invocation failed",
                tool = request.toolName,
                clientId = clientId,
                duration = duration,
                'error = result.message()
            );
            return result;
        }

        log:printInfo("MCP tool invocation completed",
            tool = request.toolName,
            clientId = clientId,
            duration = duration
        );

        return result;
    }
}
```

:::warning
Be careful about logging parameter values in production. Parameters may contain sensitive data (PII, credentials, confidential queries). Log parameter keys but redact values, or implement a data classification policy for your audit logs.
:::

## Input Validation

Validate tool parameters before executing business logic. This protects against malicious or malformed inputs from LLM clients:

```ballerina
@mcp:Tool {
    name: "queryDatabase",
    description: "Run a read-only query against the product database",
    parameters: {
        query: {type: "string", description: "SQL SELECT query", required: true}
    }
}
resource function post queryDatabase(string query) returns json|error {
    // Validate input before execution
    if !query.toLowerAscii().startsWith("select") {
        return error("Only SELECT queries are allowed");
    }
    if re`(drop|delete|update|insert|alter|truncate)`.find(query.toLowerAscii()) is re:Span {
        return error("Destructive SQL operations are not permitted");
    }
    return executeQuery(query);
}
```

## Security Checklist

Use this checklist when deploying MCP servers to production:

- [ ] Authentication enabled (bearer token or OAuth 2.0)
- [ ] Rate limiting configured per tool based on cost
- [ ] Audit logging active with sensitive data redaction
- [ ] Input validation on all tool parameters
- [ ] Destructive operations (write/delete) require additional confirmation
- [ ] HTTPS enabled for SSE transport
- [ ] API keys and secrets stored in [secrets management](/docs/deploy-operate/secure/secrets-encryption)
- [ ] Tool descriptions reviewed to avoid leaking internal implementation details

For production deployment and infrastructure security, see [Deploy & Operate](/docs/deploy-operate).

## What's Next

- [MCP Overview](overview.md) — Review MCP concepts and architecture
- [Expose Integrations as MCP Servers](exposing-mcp-servers.md) — Build MCP servers for your services
- [API Security](/docs/deploy-operate/secure/api-security) — Broader security patterns for your integrations
