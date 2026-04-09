---
title: "What Are Tools?"
description: "Understand how tools give AI agents the ability to interact with the real world beyond conversation."
---

# What Are Tools?

In the context of AI agents, a **tool** is a function that the agent can choose to invoke during its reasoning process. Tools extend an agent's capabilities beyond text generation, allowing it to query databases, call APIs, send messages, create records, or perform any action your integration needs.

Without tools, an agent can only generate text based on its training data and the conversation context. With tools, an agent becomes an active participant in your system -- it can look up real data, trigger workflows, and produce responses grounded in live information.

## How Tool Calling Works

When an agent has tools available, the following happens during the reasoning loop:

1. **Tool discovery** -- When the agent is created, it receives descriptions of all available tools, including their names, parameter schemas, and descriptions of what they do.
2. **Decision** -- For each user message, the LLM decides whether to respond directly or call one or more tools. This decision is based on the user's intent and the tool descriptions.
3. **Invocation** -- If the LLM chooses to call a tool, it generates the tool name and arguments in a structured format. The runtime executes the function with those arguments.
4. **Result processing** -- The tool's return value is sent back to the LLM, which incorporates the result into its reasoning and either responds to the user or calls additional tools.

```
User: "What meetings do I have tomorrow?"
    |
    v
LLM reasons: "I need to check the calendar"
    |
    v
Calls tool: listCalendarEvents(date = "2026-03-12")
    |
    v
Tool returns: [{title: "Sprint Review", time: "2:00 PM"}, ...]
    |
    v
LLM generates: "You have one meeting tomorrow: Sprint Review at 2:00 PM."
```

A single user message can trigger multiple tool calls. The agent might call one tool, examine the results, then call another tool based on what it learned. This iterative process continues until the agent has enough information to respond.

## What Makes a Good Tool

Effective tools share these characteristics:

- **Clear name** -- The tool name should describe what it does. `listUnreadEmails` is better than `getStuff`.
- **Descriptive parameters** -- Each parameter should have a meaningful name and type. The LLM uses these to decide what arguments to pass.
- **Typed return values** -- Structured return types help the LLM interpret results accurately.
- **Focused scope** -- Each tool should do one thing well. Prefer `listEmails` and `sendEmail` over a single `manageEmails` tool with a mode parameter.

## Tools in Ballerina

In WSO2 Integrator, you create tools by annotating functions with `@ai:AgentTool`. The annotation tells the runtime to expose the function to the agent as a callable tool:

```ballerina
import ballerina/ai;
import ballerina/io;

@ai:AgentTool
isolated function addTask(string description) returns string {
    // tool implementation
    return "Task created";
}

@ai:AgentTool
isolated function listTasks() returns Task[] {
    return tasks.toArray();
}

public function main() returns error? {
    ai:Agent agent = check new ("gpt-4-turbo");
    string response = check agent.runWithTools("Show me my tasks");
    io:println(response);
}
```

The `@ai:AgentTool` annotation automatically:

- Extracts the function name and uses it as the tool name.
- Generates a parameter schema from the function signature.
- Makes the function available to the agent during reasoning.

## Sources of Tools

Tools can come from several places in WSO2 Integrator:

| Source | Description |
|---|---|
| **Local functions** | Ballerina functions annotated with `@ai:AgentTool` |
| **Connector actions** | Actions from prebuilt connectors (Gmail, Slack, Google Calendar, databases) exposed as tools |
| **MCP servers** | Tools discovered from external MCP-compatible services |
| **OpenAPI specifications** | Tools auto-generated from API specs |

This flexibility means you can give an agent access to virtually any service or API without writing custom integration code for each one.

## Tool Calling and Safety

Because tools can perform real actions (sending emails, creating records, modifying data), safety is an important consideration:

- **Confirmation prompts** -- For destructive or irreversible actions, instruct the agent (via the system prompt) to confirm with the user before executing.
- **Parameter validation** -- The runtime validates tool arguments against the function signature before execution.
- **Error handling** -- If a tool call fails, the error is returned to the LLM, which can decide how to proceed (retry, ask the user for clarification, or report the error).

## What's Next

- [What Is an AI Agent?](what-is-an-ai-agent.md) -- How agents use tools within the perceive-reason-act loop
- [What Is MCP?](what-is-mcp/index.md) -- A standard protocol for tool discovery and invocation across systems
- [What Is Memory?](what-is-memory.md) -- How agents maintain context alongside tool use
