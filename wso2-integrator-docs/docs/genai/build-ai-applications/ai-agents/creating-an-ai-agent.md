---
sidebar_position: 1
title: "Creating an AI Agent"
description: Learn how to create AI agents in WSO2 Integrator — chat agents with built-in REST endpoints and inline agents embedded in existing services.
---

# Creating an AI Agent

Agents in WSO2 Integrator are autonomous components that perceive user input, reason about what to do, and act by calling tools or generating responses. They are backed by LLMs for decision-making and follow a **perceive-reason-act** loop.

## The Agent Loop

Every agent follows this loop:

1. **Perceive** -- The agent receives input (a user message, an API request, or an event).
2. **Reason** -- The LLM processes the input along with conversation history, system instructions, and available tool descriptions. It decides whether to respond directly or call one or more tools.
3. **Act** -- The agent either returns a response to the user or executes tool calls. If tools were called, their results feed back into the reasoning step for another iteration.

```
+-----------------------------------------+
|              Agent Loop                 |
|                                         |
|   User Input --> Perceive               |
|                     |                   |
|                  Reason (LLM)           |
|                   /     \               |
|            Respond     Call Tools       |
|               |           |             |
|            Output    Tool Results --+   |
|                                     |   |
|                  Reason (LLM) <-----+   |
|                     |                   |
|                  Respond                |
|                     |                   |
|                  Output                 |
+-----------------------------------------+
```

The loop continues until the LLM decides it has enough information to respond, or until a maximum iteration limit is reached.

:::info
The agent loop is not a simple request-response cycle. A single user message can trigger multiple rounds of reasoning and tool calling before a final answer is produced.
:::

## Agents vs. Direct LLM Calls

| Capability | Direct LLM Call | Agent |
|---|---|---|
| Single question/answer | Yes | Overkill |
| Multi-turn conversation | Manual | Built-in |
| Tool calling | Manual orchestration | Automatic |
| Memory management | Manual | Configurable |
| Streaming responses | Yes | Yes |
| Autonomous decision-making | No | Yes |

Use a **direct LLM call** when you need a one-shot transformation or classification. Use an **agent** when the task requires reasoning, tool use, or multi-turn interaction.

## Agent Types

WSO2 Integrator supports two primary agent types: **chat agents** and **inline agents**.

### Chat Agents

Chat agents are user-facing agents exposed through a REST API with built-in chat semantics. They are ideal for conversational interfaces -- chatbots, support agents, personal assistants.

Key characteristics:

- Automatic REST endpoint with `/chat` resource via `ai:Listener`
- Built-in chat interface for testing
- Conversation memory handled automatically
- Streaming response support

The following example shows a chat agent exposed as an HTTP service using `ai:Listener`:

```ballerina
import ballerina/ai;
import ballerina/http;

service /tasks on new ai:Listener(8080) {
    resource function post chat(@http:Payload ai:ChatReqMessage request) returns ai:ChatRespMessage|error {
        string response = check taskAssistantAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}

final ai:Agent taskAssistantAgent = check new ({
    systemPrompt: {
        role: "Task Assistant",
        instructions: string `You are a helpful assistant for managing a to-do list.`
    },
    tools: [addTask, listTasks, getCurrentDate],
    model: check ai:getDefaultModelProvider()
});
```

### Inline Agents (API-Exposed)

Inline agents are embedded directly into your service logic -- inside HTTP handlers, GraphQL resolvers, or automation flows. They do not expose their own endpoint; instead, you call them programmatically.

Key characteristics:

- No dedicated endpoint -- lives inside existing service logic
- Fine-grained control over when and how the agent runs
- Suitable for GraphQL resolvers, webhook handlers, and automation steps
- Stateless by default

| Factor | Chat Agent | Inline Agent |
| --- | --- | --- |
| Exposed as a REST API | Yes -- automatic endpoint | No -- you call it from code |
| Built-in chat interface | Yes | No |
| Built-in memory / sessions | Yes | Manual (stateless by default) |
| Embedded in existing services | No | Yes |
| Best for | User-facing conversations | Programmatic, single-task AI calls |

:::tip
Use a **chat agent** when you need an interactive conversation with memory. Use an **inline agent** when you need to run a one-shot AI task inside existing service logic -- summarization, moderation, classification, or tool-augmented reasoning.
:::

## Creating a Basic Agent

### Step 1: Create a New Integration Project

1. Click the **WSO2 Integrator: BI** icon in the sidebar.
2. Click the **Create New Integration** button.
3. Enter the project name.
4. Select the project directory location by clicking the **Select Location** button.
5. Click the **Create New Integration** button to generate the integration project.

### Step 2: Create an Agent

1. Click the **+** button on the WSO2 Integrator: BI side panel, or navigate back to the design screen and click **Add Artifact**.
2. Select **AI Chat Agent** under the **AI Agent** artifacts.
3. Provide a **Name** for the agent. It will take a moment to create an agent with the default configuration.
4. After creating the agent, you can configure it with a model provider, memory, tools, roles, and instructions.

### Step 3: Configure Agent Behavior

The agent's role and instructions are the most important settings. They define the agent's persona, set behavioral boundaries, and tell it when and how to use tools.

1. Click the **AI Agent** box to open the agent configuration settings.
2. Define the agent's **Role** and provide **Instructions** in natural language.
3. Click **Save** to finalize.

:::tip
Write instructions the way you would brief a new team member -- be specific about tone, constraints, and edge cases. The more precise your instructions, the more predictable the agent's responses will be.
:::

### Step 4: Configure the Model Provider

By default, the AI agent is configured to use the **Default Model Provider (WSO2)**, which uses a WSO2-hosted LLM. To use this provider, you must sign in to **BI Copilot**. Configure it as follows:

- Press `Ctrl/Cmd` + `Shift` + `P` to open the VS Code Command Palette.
- Run the command: **`Ballerina: Configure default WSO2 model provider`**.

#### Use a Different Model Provider

If you want to use a different model provider (for example, OpenAI):

1. Locate the circle with the **WSO2 logo** connected to the **AI Agent** box.
2. Click the circle to open the model configuration options.
3. Click **Create New Model Provider**.
4. Select the desired model provider from the list (e.g., **OpenAI Model Provider**).
5. Configure the model provider with the required details. Use the **Expression Helper** to reference configurables for API keys.
6. Select the desired model type and click **Save**.

:::warning
Always externalize API keys using configurable values. Hardcoding secrets in your source files risks committing them to version control. Use the Expression Helper to reference configurables.
:::

## Agent with Local Tools Example

The following example demonstrates a complete agent with locally defined tools for task management:

```ballerina
import ballerina/ai;
import ballerina/io;
import ballerina/time;
import ballerina/uuid;

type Task record {|
    string description;
    time:Date dueBy?;
    time:Date createdAt = time:utcToCivil(time:utcNow());
    time:Date completedAt?;
    boolean completed = false;
|};

isolated map<Task> tasks = {};

@ai:AgentTool
isolated function addTask(string description, time:Date dueBy?) returns string {
    string taskId = uuid:createType4AsString();
    tasks[taskId] = {description, dueBy, createdAt: time:utcToCivil(time:utcNow()), completed: false};
    return string `Task created with ID: ${taskId}`;
}

@ai:AgentTool
isolated function listTasks() returns Task[] {
    return tasks.toArray();
}

@ai:AgentTool
isolated function completeTask(string taskId) returns string {
    if tasks.hasKey(taskId) {
        tasks[taskId].completed = true;
        tasks[taskId].completedAt = time:utcToCivil(time:utcNow());
        return string `Task ${taskId} marked as completed`;
    }
    return string `Task ${taskId} not found`;
}

public function main() returns error? {
    ai:Agent agent = check new ("gpt-4-turbo");
    string response = check agent.runWithTools("Show me my tasks and mark the first one as done");
    io:println(response);
}
```

## Interacting with the Agent

WSO2 Integrator provides a built-in chat interface so you can test your agent without leaving the IDE.

1. Click the **Chat** button at the top-left corner.
2. Click **Run Integration**.

The chat panel opens and your agent is ready for conversation.

## Debugging Agent Responses with Tracing

When an agent response is not what you expect, tracing lets you inspect every step of the agent loop -- LLM calls, tool invocations, and intermediate reasoning.

1. Press `Ctrl/Cmd` + `Shift` + `P` to open the VS Code Command Palette.
2. Run the command: **`Ballerina: Enable Tracing`**.
3. Click the **Chat** button and **Run Integration**.
4. Interact with the agent.
5. Click the **Show Logs** button under the response to view the detailed trace.

:::tip
You can also integrate with external observability platforms like **Jaeger** for production-grade distributed tracing across your entire integration flow.
:::

## What's Next

- [Adding Tools to an Agent](adding-tools-to-an-agent.md) -- Connect your agent to external services, connectors, and APIs
- [Adding Memory to an Agent](adding-memory-to-an-agent.md) -- Choose between conversation-window, semantic, and persistent memory strategies
- [Agent Configuration Options](agent-configuration-options.md) -- Multi-agent orchestration and advanced configuration
