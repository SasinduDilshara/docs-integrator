---
sidebar_position: 5
title: AI Agent Observability
description: Monitor and trace AI agent behavior with OpenTelemetry tracing, structured logging, and metrics.
---

# AI Agent Observability

Observability gives you visibility into how your AI agents reason, which tools they call, how long each step takes, and where failures occur. Without it, debugging agent behavior in production is guesswork.

WSO2 Integrator supports OpenTelemetry-based tracing and metrics for agents through Ballerina's standard observability stack. You do not need any agent-specific observability API -- the runtime wraps `agent.run(...)` calls, tool invocations, and LLM calls in spans automatically once observability is enabled.

## Enabling Observability

Turn on observability in `Ballerina.toml`:

```toml
[build-options]
observabilityIncluded = true
```

This bundles the Ballerina observability runtime with your package. Then configure a tracing provider and (optionally) a metrics reporter in `Config.toml`:

```toml
[ballerina.observe]
tracingEnabled = true
tracingProvider = "jaeger"
metricsEnabled = true
metricsReporter = "prometheus"
```

Any OpenTelemetry-compatible backend (Jaeger, Zipkin, Honeycomb, Grafana Tempo, and so on) works. Consult the Ballerina documentation for the provider-specific configuration blocks.

### Trace Structure

Once tracing is enabled, every call to `agent.run(...)` produces a trace with a span hierarchy similar to the one below. Tool invocations and LLM calls appear as child spans.

```
agent.run
  |-- llm.call (system prompt + user message)
  |-- tool.execute (getCustomer)
  |     |-- http.get /customers/C-1234
  |-- llm.call (with tool result)
  |-- tool.execute (searchOrders)
  |     |-- http.get /orders?customerId=C-1234
  |-- llm.call (final response generation)
```

Ballerina's HTTP client instrumentation automatically adds spans for any `http:Client` calls made inside your tools, so you get end-to-end traces from the user request down to your backends.

## Structured Logging

Use the standard `ballerina/log` module to emit structured logs around agent calls. The `log` module supports key-value pairs that most log aggregators can index.

```ballerina
import ballerina/ai;
import ballerina/log;

service /support on new ai:Listener(8090) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        log:printInfo("Agent invoked",
            sessionId = request.sessionId,
            userMessage = request.message
        );

        string response = check supportAgent.run(request.message, request.sessionId);

        log:printInfo("Agent responded",
            sessionId = request.sessionId,
            responseLength = response.length()
        );

        return {message: response};
    }
}
```

Tools can log their own events the same way. Because tool functions are `isolated`, prefer `log:printInfo` / `log:printError` over custom loggers that require shared state.

```ballerina
import ballerina/ai;
import ballerina/log;

# Create a new support ticket.
# + subject - Ticket subject
# + description - Ticket description
# + return - The created ticket
@ai:AgentTool
isolated function createSupportTicket(string subject, string description)
        returns json|error {
    log:printInfo("Creating support ticket", subject = subject);
    json result = check ticketApi->/tickets.post({subject, description});
    log:printInfo("Support ticket created", ticketId = result.toString());
    return result;
}
```

## Metrics

With `metricsEnabled = true`, Ballerina publishes standard runtime and HTTP metrics automatically. For domain-specific monitoring -- for example, the number of tickets the agent opens per hour -- use `ballerina/observe` counters and gauges.

```ballerina
import ballerina/ai;
import ballerina/observe;

final observe:Counter ticketCreations = new (
    "agent_tickets_created_total",
    desc = "Number of support tickets created by the agent"
);

# Create a new support ticket.
# + subject - Ticket subject
# + description - Ticket description
# + return - The created ticket
@ai:AgentTool
isolated function createSupportTicket(string subject, string description)
        returns json|error {
    json result = check ticketApi->/tickets.post({subject, description});
    ticketCreations.increment();
    return result;
}
```

### Exporting to Prometheus

```toml
[ballerina.observe]
metricsEnabled = true
metricsReporter = "prometheus"

[ballerinax.prometheus]
port = 9797
host = "0.0.0.0"
```

### Example Grafana Queries

Once metrics land in Prometheus you can build dashboards from them. The exact metric names depend on the Ballerina runtime version, but common queries look like this:

**95th-percentile HTTP request duration (covers the `ai:Listener` service):**

```promql
histogram_quantile(0.95, rate(response_time_seconds_bucket[5m]))
```

**Custom ticket creation rate:**

```promql
rate(agent_tickets_created_total[5m])
```

## WSO2 Integrator Console

When you deploy agents to the WSO2 Integrator Console (Choreo), traces and metrics are collected automatically. The console provides built-in views for request rates, latencies, and distributed traces so you can inspect tool calls and LLM interactions without wiring up your own backend. Use the same `log`, `observe`, and tracing APIs described above -- the console consumes the data without additional configuration.

## Debugging Agent Behavior

### Verbose Development Logs

During development, lower the log level to capture everything the agent and its tools emit. Add this to `Config.toml`:

```toml
[ballerina.log]
level = "DEBUG"
```

Remember to raise the level again before shipping to production, since debug-level logs may include sensitive data.

### Inspecting Tool Inputs and Outputs

Because every tool is a regular Ballerina function, the simplest way to inspect what the agent is doing is to log the arguments and return values inside the tool itself:

```ballerina
# Look up an order by ID.
# + orderId - Order identifier
# + return - Order payload
@ai:AgentTool
isolated function getOrder(string orderId) returns json|error {
    log:printDebug("Tool invoked", 'tool = "getOrder", orderId = orderId);
    json result = check orderApi->/orders/[orderId];
    log:printDebug("Tool returning", 'tool = "getOrder", result = result);
    return result;
}
```

Combined with distributed traces, these logs make it straightforward to follow the agent's reasoning step by step.

## What's Next

- [AI Agent Evaluations](/docs/genai/develop/agents/agent-evaluations) -- Test and measure agent quality
- [Creating an AI Agent](/docs/genai/develop/agents/creating-agent) -- Build your first agent
- [Advanced Configuration](/docs/genai/develop/agents/advanced-config) -- Tune agent behavior
