---
sidebar_position: 2
title: "Adding Tools to an Agent"
description: Bind tools to agents using the @ai:AgentTool annotation, connector actions, and tool kits to let agents interact with external services.
---

# Adding Tools to an Agent

Tools give your agent the ability to interact with the real world beyond conversation. When you bind tools to an agent, the LLM decides when to call them, what arguments to pass, and how to interpret the results -- all automatically.

Tools can be created from:

- **Ballerina functions** -- Annotate with `@ai:AgentTool`
- **Connector actions** -- Expose connector methods (Gmail, Calendar, Slack, etc.) as agent tools
- **Tool kits** -- Group related tools using `ai:BaseToolKit`

## Defining Tool Functions with @ai:AgentTool

The `@ai:AgentTool` annotation marks a function as available for agent use. The LLM uses the function name, parameter names, and types to understand when and how to call the tool.

```ballerina
import ballerina/ai;

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
```

:::tip
The more descriptive your function names and parameter names are, the better the LLM will be at choosing the right tool for a given query. Use clear, action-oriented names like `addTask`, `listTasks`, or `sendEmail`.
:::

## Adding Tools via the Visual Designer

You can also add tools through the WSO2 Integrator visual designer:

### Create a Function

1. Click the **+** button in the WSO2 Integrator side panel under the **Functions** section.
2. Provide the required details -- function name, parameters, and return types.
3. Implement the function logic in the flow node editor.

### Add the Function as a Tool

1. Go to the agent flow view.
2. Click the **+** button at the bottom-right corner of the `AI Agent` box.
3. Select the **Use Function** option.
4. Select the function from the **Current Integration** list.
5. Provide the **Tool Name** and **Description** of the tool.

:::note
The more descriptive your tool descriptions, the better the LLM will be at choosing the right tool for a given query.
:::

## Using Connector Actions as Tools

WSO2 Integrator ships with prebuilt connectors for popular external services like Gmail, Google Calendar, Slack, and many more. You can directly use their actions as agent tools -- no custom integration code needed. The connector handles authentication, serialization, and API communication for you.

The following example shows an agent with Gmail and Google Calendar tools:

```ballerina
import ballerina/ai;
import ballerinax/googleapis.gmail;
import ballerinax/googleapis.calendar;

@ai:AgentTool
isolated function readUnreadEmails() returns gmail:Message[]|error {
    gmail:ListMessagesResponse messageList = check gmailClient->/users/me/messages(q = "label:INBOX is:unread");
    // ...
}

@ai:AgentTool
isolated function sendEmail(string[] to, string subject, string body) returns gmail:Message|error {
    return gmailClient->/users/me/messages/send.post({to, subject, bodyInText: body});
}

final ai:Agent personalAssistantAgent = check new (
    systemPrompt = {
        role: "Personal Assistant",
        instructions: string `You are an intelligent personal AI assistant...`
    },
    model = check ai:getDefaultModelProvider(),
    tools = [readUnreadEmails, sendEmail, getCalendarEvents, createCalendarEvent]
);
```

### Adding Connector Tools via the Visual Designer

The following steps walk through adding a Gmail connector tool:

1. **Add the connector.** Search for the desired connector (e.g., Gmail) in the connector palette and add it to your project.
2. **Configure the connector.** Select the authentication method (e.g., `OAuth2RefreshTokenGrantType`) and fill in your credentials.
3. **Create the tool.** Create a new tool and select the desired action from the connector.
4. **Customize parameters.** Set default values for parameters that should not be exposed to the agent (e.g., set `userId` to `me` for Gmail).
5. **Clean up exposed parameters.** Remove parameters that the agent does not need to set.

:::tip
You can reuse the same connector instance across multiple tools. For example, `readSpecificEmail` and `sendEmail` can both reuse the same `gmailClient` you configured once. This keeps your configuration DRY and credentials centralized.
:::

### Prerequisites for External Connectors

Before using external connectors, ensure you have the required credentials. For example, Google API connectors require:

- **Client ID**
- **Client Secret**
- **Refresh Token**

:::warning
You must enable the required API scopes in your cloud provider's console. Without the correct scopes, your connectors will fail to authenticate.
:::

## Tool Kits with ai:BaseToolKit

For related tools that share state or configuration, group them into a tool kit using `ai:BaseToolKit`:

```ballerina
public isolated class TaskManagerToolkit {
    *ai:BaseToolKit;

    public isolated function getTools() returns ai:ToolConfig[] =>
        ai:getToolConfigs([self.addTask, self.listTasks]);

    @ai:AgentTool
    isolated function addTask(string description, string? dueBy = ()) returns string {
        // ...
    }

    @ai:AgentTool
    isolated function listTasks() returns string {
        // ...
    }
}
```

Tool kits are useful when:

- Multiple tools share the same database connection or client
- Tools need access to shared state
- You want to package and reuse a set of related tools across agents

## Externalizing Credentials

Hard-coding credentials in your integration is a security risk. Use BI configurables to externalize them:

```toml
# Config.toml
[gmail]
clientId = "<your-client-id>"
clientSecret = "<your-client-secret>"
refreshToken = "<your-refresh-token>"

[gcalendar]
clientId = "<your-client-id>"
clientSecret = "<your-client-secret>"
refreshToken = "<your-refresh-token>"
```

:::warning
Never commit `Config.toml` files containing real credentials to version control. Add `Config.toml` to your `.gitignore` file to prevent accidental exposure.
:::

## What's Next

- [Adding Memory to an Agent](adding-memory-to-an-agent.md) -- Configure memory strategies for context management
- [Agent Configuration Options](agent-configuration-options.md) -- Multi-agent orchestration and advanced configuration
- [Creating an AI Agent](creating-an-ai-agent.md) -- Review the fundamentals of agent creation
