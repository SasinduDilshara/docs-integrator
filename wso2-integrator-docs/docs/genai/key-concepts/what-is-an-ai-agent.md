---
title: "What Is an AI Agent?"
description: "Understand AI agents -- autonomous components that perceive, reason, and act using LLMs, tools, and memory."
---

# What Is an AI Agent?

An AI agent is an autonomous component that uses a Large Language Model to perceive input, reason about what to do, and take action. Unlike a simple LLM call that produces a single response, an agent can make decisions, call tools, and iterate through multiple reasoning steps before delivering a final answer.

## The Perceive-Reason-Act Loop

Every AI agent operates on a continuous loop:

1. **Perceive** -- The agent receives input. This could be a user message, an API request, or an event from an external system.
2. **Reason** -- The LLM processes the input along with conversation history, system instructions, and descriptions of available tools. It decides whether to respond directly or take an action.
3. **Act** -- The agent either returns a response to the user or executes one or more tool calls. If tools were called, their results feed back into the reasoning step for another iteration.

```
User Input --> Perceive --> Reason (LLM) --> Respond or Call Tools
                                                    |
                                              Tool Results
                                                    |
                                              Reason (LLM) --> Respond
```

The loop continues until the LLM decides it has enough information to produce a final response, or until a maximum iteration limit is reached. A single user message can trigger multiple rounds of reasoning and tool calling before an answer is produced.

## Agents vs. Direct LLM Calls

Not every LLM interaction needs an agent. Understanding when to use each approach helps you build simpler, more efficient integrations.

| Capability | Direct LLM call | Agent |
|---|---|---|
| Single question/answer | Yes | Overkill |
| Multi-turn conversation | Manual | Built-in |
| Tool calling | Manual orchestration | Automatic |
| Memory management | Manual | Configurable |
| Streaming responses | Yes | Yes |
| Autonomous decision-making | No | Yes |

Use a **direct LLM call** or a **natural function** when you need a one-shot transformation, classification, or generation. Use an **agent** when the task requires reasoning over multiple steps, calling external tools, or maintaining context across a conversation.

## Key Components of an Agent

An agent is composed of four main parts:

### LLM (the brain)

The Large Language Model provides the reasoning capability. It interprets user intent, decides which tools to call, and generates responses. The choice of model affects the agent's reasoning quality, speed, and cost.

### Tools (the hands)

Tools are functions the agent can invoke to interact with the outside world -- query databases, call APIs, send emails, search the web. The LLM reads tool descriptions and decides when and how to call them. Without tools, an agent can only generate text; with tools, it can take real action.

### Memory (the context)

Memory determines what the agent remembers across interactions. This includes recent messages (conversation memory), semantically relevant past interactions (semantic memory), and persistent storage across sessions (persistent memory). Memory gives the agent context to produce relevant, coherent responses.

### System prompt (the personality)

The system prompt defines the agent's role, behavior guidelines, and constraints. It shapes how the agent responds, what tone it uses, and what boundaries it respects. A well-crafted system prompt is the difference between a generic chatbot and a specialized assistant.

## Creating an Agent in Ballerina

WSO2 Integrator lets you create agents with tools bound directly to them. Here is a minimal example:

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

In this example:

- Functions annotated with `@ai:AgentTool` become tools the agent can call.
- The agent is created with a model identifier.
- When `runWithTools` is called, the agent enters the perceive-reason-act loop: it reads the user message, sees the available tools, decides to call `listTasks`, processes the results, and generates a response.

## What's Next

- [What Are Tools?](what-are-tools.md) -- How tools extend agent capabilities
- [What Is Memory?](what-is-memory.md) -- How agents maintain context across interactions
- [What Is a Large Language Model?](what-is-an-llm.md) -- The LLMs that power agent reasoning
