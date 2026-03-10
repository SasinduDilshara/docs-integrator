---
sidebar_position: 3
title: "API-Exposed Agents (Inline Agents)"
---

When you build a [chat agent](chat-agents.md), WSO2 Integrator automatically exposes it as a REST API endpoint with a built-in chat interface, session management, and memory. That is perfect for user-facing conversations — but it is not the only way to use agents.

**Inline agents** flip the model. Instead of living behind their own API endpoint, they are invoked programmatically from anywhere within your integration logic — inside an HTTP resource, a GraphQL resolver, an event handler, or any other service function. Think of them as on-demand AI helpers you call like a regular function.

| Factor | Chat Agent | Inline Agent |
| --- | --- | --- |
| Exposed as a REST API | Yes — automatic endpoint | No — you call it from code |
| Built-in chat interface | Yes | No |
| Built-in memory / sessions | Yes | Manual (stateless by default) |
| Embedded in existing services | No | Yes |
| Best for | User-facing conversations | Programmatic, single-task AI calls |

:::tip
Use a **chat agent** when you need an interactive conversation with memory. Use an **inline agent** when you need to run a one-shot AI task inside existing service logic — summarization, moderation, classification, or tool-augmented reasoning.
:::

In this guide, you will build a **GraphQL service** with a mutation that invokes an inline agent. By the end, you will have a working GraphQL endpoint that accepts a natural-language query, routes it through an AI agent, and returns the result — all without exposing a separate REST endpoint.

---

## Step 1: Create a new integration project

Start by scaffolding a new project to hold your GraphQL service and inline agent.

1. Click the **WSO2 Integrator: BI** icon in the sidebar.
2. Click the **Create New Integration** button.
3. Enter the project name as `GraphqlService`.
4. Select the project directory location and click **Create New Integration**.

![Create a new integration project](/img/genai/agents/inline-agents/inline-agent-create-a-new-integration-project.gif)

## Step 2: Create a GraphQL service

With your project ready, add a GraphQL service artifact.

1. In the design view, click **Add Artifact**.
2. Select **GraphQL Service** under **Integration as API**.
3. Keep the default base path and port, then click **Create**.

![Create a GraphQL service](/img/genai/agents/inline-agents/inline-agent-create-a-graphql-service.gif)

:::info
A GraphQL service lets you define strongly-typed queries and mutations. Clients send a single request and receive exactly the data they ask for — making it a natural fit for AI-powered fields where the response shape matters.
:::

## Step 3: Create a GraphQL resolver

Now define the mutation that will invoke your inline agent.

1. Click **+ Create Operations**.
2. In the side panel, click **+** in the **Mutation** section.
3. Set the field name to `task`.
4. Add an argument with the name `query` and the type `string`.
5. Set the field type to `string|error`.

![Create a GraphQL resolver](/img/genai/agents/inline-agents/inline-agent-create-a-graphql-resolver.gif)

## Step 4: Implement the resolver with an inline agent

This is where the magic happens. Instead of writing manual business logic, you drop an AI agent directly into the resolver flow.

1. Click the `task` operation to open the resolver editor.
2. Click **+** in the flow and select **Agent** under **AI**.
3. Update the **Role** and **Instructions** fields to describe what the agent should do (for example, "You are a task assistant that summarizes and acts on user requests").
4. Switch the **Query** field to **Expression** mode and provide the `query` parameter so the agent receives the user's input.
5. Set the **Result** variable to `response` and click **Save**.

![Implement the resolver with an inline agent](/img/genai/agents/inline-agents/inline-agent-implement-resolver.gif)

6. Configure **memory**, **model**, and **tools** as needed. The configuration steps are the same as for a chat agent — refer to the [Create a Chat Agent](chat-agents.md) guide for detailed instructions on setting up model providers, memory strategies, and tool bindings.

:::note
You must implement at least one **query** operation for a valid GraphQL service. Add a simple `greet` query that returns `"welcome"` to satisfy this requirement.
:::

## Step 5: Add a return node

After the agent processes the request, you need to return its output to the GraphQL client.

1. Click **+** below the Agent node in the flow.
2. Add a **Return** node.
3. Provide the `response` variable as the return value.

![Return the agent response](/img/genai/agents/inline-agents/inline-agent-return-response.gif)

## Step 6: Run and query the service

You are ready to test. Run the service and send a GraphQL mutation to your inline agent.

1. Click **Run** to start the integration.
2. Open a terminal and send a query with curl:

```bash
curl -X POST http://localhost:8080/graphql \
  -H "Content-Type: application/json" \
  -d '{ "query": "mutation Task { task(query: \"Summarize latest emails\") }" }'
```

The agent processes the natural-language query, invokes any configured tools, and returns the result through the GraphQL response.

![Run and query the service](/img/genai/agents/inline-agents/inline-agent-execute.gif)

:::tip
You can test GraphQL services with any GraphQL client — Postman, Insomnia, or GraphQL Playground — not just curl. Point the client at `http://localhost:8080/graphql` and use the mutation schema to explore available operations.
:::

---

## Key takeaways

- **Chat agents** are REST APIs with built-in session management and a chat interface. They are ideal for multi-turn, user-facing conversations.
- **Inline agents** are not tied to any endpoint. You invoke them programmatically from inside your integration logic — GraphQL resolvers, HTTP resources, event handlers, or any other service function.
- Inline agents are **stateless by default**. If you need conversational context across calls, pass the history explicitly or use a chat agent instead.
- You configure inline agents (model, memory, tools) the same way you configure chat agents. The only difference is *where* and *how* you invoke them.

## What's next

You now have a working GraphQL service powered by an inline agent. Here are some natural next steps:

- [Configure Agent Memory](memory-configuration.md) — Set up memory strategies for chat agents and stateful inline agents
- [Bind Tools to Agents](tool-binding.md) — Give your agent the ability to call external services, connectors, and APIs
- [Orchestrate Multiple Agents](multi-agent-orchestration.md) — Route between specialized agents for complex workflows
