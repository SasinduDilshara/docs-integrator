---
sidebar_position: 5
title: Observability
description: Reference for tracing, logging, and metrics on AI agents in WSO2 Integrator — Tracing toggle, OpenTelemetry, and Prometheus integration.
---

# Observability

Without observability, debugging an AI agent in production is guesswork. WSO2 Integrator wraps every `agent.run`, every tool invocation, and every LLM call in OpenTelemetry spans automatically — so you get end-to-end traces of *exactly* what the agent did.

## Turning Tracing On

The fastest way to enable tracing is the **Tracing** toggle at the top right of the agent canvas.

- **Tracing: Off** — agent runs do not produce spans (default for new agents during development).
- **Tracing: On** — every run produces a span tree. Spans are emitted to whatever OpenTelemetry backend the project is configured with.

The toggle is project-level: turn it on once and every agent in the project starts emitting traces.

For a deployable build, observability is configured in two files:

```toml
# Ballerina.toml
[build-options]
observabilityIncluded = true
```

```toml
# Config.toml
[ballerina.observe]
tracingEnabled = true
tracingProvider = "jaeger"
metricsEnabled = true
metricsReporter = "prometheus"
```

Any OpenTelemetry-compatible backend works — Jaeger, Zipkin, Honeycomb, Grafana Tempo, the Choreo console — by changing `tracingProvider` and adding the provider's configuration block.

## What a Trace Looks Like

A single call to `agent.run(...)` produces a hierarchy of spans:

```bash
agent.run
  ├── llm.call               (system prompt + user message)
  ├── tool.execute           (getCustomer)
  │     └── http.get         /customers/C-1234
  ├── llm.call               (with tool result)
  ├── tool.execute           (searchOrders)
  │     └── http.get         /orders?customerId=C-1234
  └── llm.call               (final response generation)
```

Each span carries:

- **Name** of the operation (`tool.execute`, `llm.call`, …).
- **Duration** — how long the LLM/tool/HTTP call took.
- **Attributes** — model name, tool name, HTTP status, error, etc.
- **Events** — important checkpoints, such as the moment the LLM picked a tool.

Ballerina's HTTP client is instrumented automatically, so HTTP calls inside your tools appear as child spans without you doing anything.

## Logging

Use the standard `ballerina/log` module to emit structured logs. The module supports key-value pairs that any log aggregator can index.

```ballerina
import ballerina/ai;
import ballerina/log;

service /support on new ai:Listener(8090) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        log:printInfo("Agent invoked",
            sessionId = request.sessionId,
            messageLength = request.message.length()
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

Tools can log inside their bodies the same way. Combined with traces, these logs make it straightforward to follow the agent's reasoning step by step.

For development, lower the log level in `Config.toml`:

```toml
[ballerina.log]
level = "DEBUG"
```

> Remember to raise the level back before shipping — debug logs may include the full LLM prompt, which can contain sensitive data.

## Metrics

With `metricsEnabled = true`, Ballerina publishes standard runtime and HTTP metrics automatically. For domain-specific monitoring use the `ballerina/observe` module:

```ballerina
import ballerina/ai;
import ballerina/observe;

final observe:Counter ticketsCreated = new (
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
    ticketsCreated.increment();
    return result;
}
```

For Prometheus:

```toml
[ballerina.observe]
metricsEnabled = true
metricsReporter = "prometheus"

[ballerinax.prometheus]
port = 9797
host = "0.0.0.0"
```

Sample dashboard queries:

```promql
# 95th-percentile request duration over the last 5 minutes
histogram_quantile(0.95, rate(response_time_seconds_bucket[5m]))

# Tickets the agent opens per minute
rate(agent_tickets_created_total[1m])
```

## The WSO2 Integrator Console (Choreo)

When you deploy agents to the WSO2 Integrator Console, traces and metrics are collected automatically. The console provides built-in views for request rates, latencies, and distributed traces — you can inspect tool calls and LLM interactions without standing up your own backend. Use the same `log`, `observe`, and tracing APIs above; the console picks them up.

## Debugging Agent Behaviour

The most useful technique once tracing is on:

1. Trigger the agent with a problematic input.
2. Open the trace in your backend.
3. Look at the sequence of `tool.execute` spans — did the agent pick the right tools, in the right order?
4. Look at the `llm.call` spans — what did the model see at each step?

If the agent is picking the wrong tool, the answer is almost always in the tool descriptions or the system prompt. If the agent's *final* answer is wrong but the tools were called correctly, the answer is in the LLM's interpretation of the tool results.

Both of those are visible in the trace. You don't need to attach a debugger or print everything by hand.

## What's Next

- **[Evaluations](evaluations.md)** — protect against regressions automatically.
- **[Tools](tools.md)** — well-described tools are the biggest lever on observability quality.
- **[Memory](memory.md)** — every turn's memory contents appear in the LLM call spans.
