---
title: "Build a Smart Calculator Assistant"
description: Build a math tutor agent step by step with arithmetic tools, memory, and tracing in WSO2 Integrator.
---

# Build a Smart Calculator Assistant

In this tutorial, you will build a **Math Tutor** chat agent step by step. By the end, you will have a working agent that can perform arithmetic operations, hold multi-turn conversations, and produce traceable responses -- all without writing boilerplate HTTP or session-management code.

Chat agents are the primary way to build user-facing conversational experiences in WSO2 Integrator. You define a role, connect an LLM, configure memory, and bind tools -- then WSO2 Integrator handles the REST endpoint, session management, and chat interface for you.

## Step 1: Create a new integration project

Start by scaffolding a new project that will hold your agent and its supporting artifacts.

1. Click the **WSO2 Integrator: BI** icon in the sidebar.
2. Click the **Create New Integration** button.
3. Enter the project name as `MathTutor`.
4. Select the project directory location by clicking the **Select Location** button.
5. Click the **Create New Integration** button to generate the integration project.

![Create a new integration project](/img/genai/agents/chat-agents/1.create-new-project.gif)

## Step 2: Create an agent

With your project ready, add a chat agent artifact.

1. Click the **+** button on the WSO2 Integrator: BI side panel, or navigate back to the design screen and click **Add Artifact**.
2. Select **AI Chat Agent** under the **AI Agent** artifacts.
3. Provide a **Name** for the agent. It will take a moment to create an agent with the default configuration.
4. After creating the agent, you can configure it with a model provider, memory, tools, roles, and instructions.

![Create an agent](/img/genai/agents/chat-agents/2.create-an-agent.gif)

## Step 3: Configure the agent behavior

The agent's role and instructions are the most important settings you will configure. They define the agent's persona, set behavioral boundaries, and tell it when and how to use tools.

1. Click the **AI Agent** box to open the agent configuration settings.
2. Define the agent's **Role** and provide **Instructions** in natural language. These instructions will guide the agent's behavior and tasks.
3. Click **Save** to finalize and complete the agent behavior configuration.

![Configure the agent behavior](/img/genai/agents/chat-agents/3.configure-behaviour.gif)

:::tip
Write instructions the way you would brief a new team member -- be specific about tone, constraints, and edge cases. The more precise your instructions, the more predictable the agent's responses will be.
:::

## Step 4: Configure the agent model

By default, the AI agent is configured to use the **Default Model Provider (WSO2)**, which uses a WSO2-hosted LLM. To use this provider, you must sign in to **BI Copilot**. When creating the agent, you will be prompted to sign in to BI Copilot. After signing in, configure the default model provider as follows:

- Press `Ctrl/Cmd` + `Shift` + `P` to open the VS Code Command Palette.
- Run the command: **`Ballerina: Configure default WSO2 model provider`**.

![Configure model provider](/img/genai/agents/chat-agents/4.configure-model-provider.png)

### Use a different model provider

If you want to use a different model provider -- for example, **OpenAI** -- follow these steps:

1. Locate the circle with the **WSO2 logo** connected to the **AI Agent** box.
2. Click the circle to open the model configuration options.
3. Click **Create New Model Provider**.
4. Select **OpenAI Model Provider** from the list.
5. Configure the model provider with the required details:
   - Switch the **API Key** field from **Text** mode to **Expression** mode using the toggle.
   - Click the **API Key** input field to open the **Expression Helper**.
   - In the **Expression Helper**, select **Configurables**.
   - Click **+ New Configurable** to define a new configurable.
   - Set the **Name** to `openAiApiKey` and the **Type** to `string`.
   - Click **Save**.
6. In the **Model Type** dropdown, select `gpt-4.1`.
7. Click **Save** to complete the LLM configuration.

![Configure the agent model](/img/genai/agents/chat-agents/4.configure-model.gif)

:::warning
Always externalize API keys using configurable values. Hardcoding secrets in your source files risks committing them to version control. Use the Expression Helper to reference configurables, as shown above.
:::

## Step 5: Configure agent memory

Memory controls how much conversational context the agent retains across turns. Longer memory gives richer context but increases token usage.

1. By default, the agent comes preconfigured with an in-memory implementation.
2. For this tutorial, you will keep the default memory configuration.
3. If you prefer a different memory type, click **Add Memory** and select your desired option.

:::info
The default in-memory implementation stores conversation history for the duration of the running session. When the service restarts, history is cleared. For persistent or semantic memory options, see [Adding Memory to an Agent](../../build-ai-applications/ai-agents/adding-memory-to-an-agent.md).
:::

## Step 6: Add tools to the agent

Tools give your agent the ability to take action -- call functions, invoke connectors, or hit external APIs. WSO2 Integrator allows you to create tools using existing functions. It also supports automatically generating tools from connector actions or OpenAPI specifications.

For the Math Tutor, you will create arithmetic functions and wire them up as agent tools.

### Create a function

1. Click the **+** button in the WSO2 Integrator side panel under the **Functions** section.
2. Provide the required details. For this example, use `sum` as the function name, and specify the parameters and return types.
3. Implement the function logic in the flow node editor that opens.

![Create a function](/img/genai/agents/chat-agents/5.create-function.gif)

### Add the created function as a tool

1. Go to the agent flow view.
2. Click the **+** button at the bottom-right corner of the `AI Agent` box.
3. Select the **Use Function** option.
4. Select the created function from the **Current Integration** list -- in this case, `sum`.
5. Provide the **Tool Name** and **Description** of the tool.

![Create an agent tool](/img/genai/agents/chat-agents/6.create-agent-tool.gif)

:::note
Follow the steps under **Create a function** to create functions named `subtract`, `multiply`, and `divide`. Then repeat the steps under **Add the created function as a tool** to add each as tools. The more descriptive your tool descriptions, the better the LLM will be at choosing the right tool for a given query.
:::

![Add functions as tools](/img/genai/agents/chat-agents/6.add-functions.gif)

## Step 7: Interact with the agent

WSO2 Integrator provides a built-in chat interface so you can test your agent without leaving the IDE.

1. Click the **Chat** button at the top-left corner.
2. Click **Run Integration**.

The chat panel opens and your Math Tutor is ready for conversation. Try asking it to solve an arithmetic problem and watch it select the appropriate tool automatically.

![Interact with the agent](/img/genai/agents/chat-agents/7.interact-with-agent.gif)

## Step 8: Debug agent responses with tracing

When an agent response is not what you expect, tracing lets you inspect every step of the agent loop -- LLM calls, tool invocations, and intermediate reasoning.

WSO2 Integrator provides a built-in tracing capability:

1. Press `Ctrl/Cmd` + `Shift` + `P` to open the VS Code Command Palette.
2. Run the command: **`Ballerina: Enable Tracing`**.
3. Click the **Chat** button.
4. Click **Run Integration**.
5. Interact with the agent.
6. Click the **Show Logs** button under the response to view the detailed trace.

![Debug with tracing](/img/genai/agents/chat-agents/8.tracing.gif)

:::tip
You can also integrate with external observability platforms like **Jaeger** for production-grade distributed tracing across your entire integration flow.
:::

## What's Next

You now have a fully functional Math Tutor chat agent. Here are some natural next steps to explore:

- [Build a Sample Hotel Booking Agent](hotel-booking-agent.md) -- Another hands-on tutorial using AI agent patterns
- [Adding Tools to an Agent](../../build-ai-applications/ai-agents/adding-tools-to-an-agent.md) -- Connect your agent to external services, connectors, and APIs
- [Adding Memory to an Agent](../../build-ai-applications/ai-agents/adding-memory-to-an-agent.md) -- Choose between conversation-window, semantic, and persistent memory strategies
- [Creating an AI Agent](../../build-ai-applications/ai-agents/creating-an-ai-agent.md) -- Expose your agent as a programmatic HTTP/REST endpoint
