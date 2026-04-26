---
sidebar_position: 4
title: What are Tools?
description: A plain-English explanation of tools — the buttons that let an AI agent do things, not just talk.
---

# What are Tools?

A **tool** is something an [AI agent](what-is-ai-agent.md) can use to do work in the real world. Looking up an order, querying a database, sending an email, calling an API — each of those is a tool.

By itself, an LLM only produces text. Tools are how you turn that text-producer into something that can *act*. If the agent is the brain, tools are the hands.

## A Pocket Analogy

Imagine you ask a smart friend to help you handle customer support emails. They are very capable, but they are sitting in another room. You hand them a folder with three small buttons taped to it:

- A button labelled **"Look up customer by ID"**.
- A button labelled **"Find recent orders"**.
- A button labelled **"Open a support ticket"**.

Each button has a tiny note next to it explaining what it does and what to write on the slip you hand back. Your friend reads the customer's email, decides which button(s) they need, scribbles the input on a slip, and slides it under the door. Someone on your side presses the button for real and slides the result back. Your friend uses that result to write the reply.

That, in essence, is how tools work:

- The **LLM never runs the tool itself.** It just says *"call `lookupCustomer` with `customerId = 'C-1234'`"*.
- The **integration runtime** is the one that actually presses the button — calls your function, hits your API, runs your query.
- The **result** is sent back to the LLM, which uses it to decide what to do next.

This separation is important. It means the LLM can never accidentally execute arbitrary code: it can only ask for tools you explicitly handed it.

## What the LLM Sees

For each tool, the LLM sees three things:

| What | Why it matters |
|---|---|
| **Name** | Lets the LLM refer to the tool. (`lookupCustomer`) |
| **Description** | Tells the LLM *what the tool does* and *when to use it*. This is the most important part — it is essentially the "instruction sticker" on the button. |
| **Parameter shape** | Tells the LLM what inputs the tool needs, and what type each one is. |

The LLM does **not** see how the tool is implemented. It does not see your code. It only sees the description and the parameter shape — the public face of the tool. That is why writing a clear description for each tool is the single most impactful thing you can do for agent quality.

> *"Looks up a customer record by their numeric customer ID. Use this any time the user mentions a specific customer."* — clear, decisive, easy to use.
>
> *"Customer endpoint."* — almost useless. The LLM has no idea when to call this.

## What Tools Look Like Inside WSO2 Integrator

In WSO2 Integrator, tools are not a special new concept — they are just **regular functions** that you mark as available to the agent. You write them the way you write any other function: receive some inputs, do some work, return a result. The runtime takes care of:

- Building the description from your function's documentation comment.
- Building the parameter schema from the function's signature.
- Running the function safely when the agent decides to call it.
- Handing the return value back to the agent.

Most of the time, "adding a tool to my agent" means writing a normal function and adding it to the agent's tools list.

## Three Common Kinds of Tools

It helps to think of tools in three categories:

| Category | Purpose | Example |
|---|---|---|
| **Read tools** | Fetch information so the agent has facts to work with. | *Look up a customer*, *find recent orders*, *get current stock level*. |
| **Action tools** | Make a change or trigger a workflow. | *Create a ticket*, *send an email*, *cancel an order*. |
| **Connector tools** | Reuse an existing WSO2 Integrator connector (Salesforce, GitHub, Gmail, a database) by wrapping it as a tool. | *Send a Gmail message*, *create a JIRA issue*. |

Read tools are usually safe to call freely. Action tools deserve a bit more thought — for irreversible operations like deletions or payments, it is good practice to bake the *"please confirm with the user first"* rule into the tool's own description.

## Designing Good Tools

A tool that's hard for the agent to use will get used badly. Three rules cover most of the failure modes:

1. **One job per tool.** A tool that does too many things is hard for the LLM to choose. Prefer two clear tools over one mega-tool.
2. **Describe in the language of the user.** If your users say *"order"*, your tool should say *"order"*, not *"transactionRecord"*.
3. **Return small, useful results.** A tool that returns a 50-page document forces the LLM to wade through it. Trim to what's relevant — names, IDs, statuses, totals.

If you are wondering how to design a particular tool, ask yourself: *"If I gave this description and this output to a sharp intern who didn't know my system, could they use it correctly?"* That is exactly the situation the LLM is in.

## Where Tools Come From

Tools do not have to live inside your own codebase. Three sources are common:

- **Your own functions** — the most common case. You write the tool right next to the rest of your integration.
- **Existing WSO2 Integrator connectors** — wrap a Salesforce, Gmail, or database call as a tool with a couple of lines.
- **MCP servers** — tools that someone else (or you) published over a standard protocol. The agent can use them without any custom code on your side. See [What is MCP?](what-is-mcp.md).

Mix and match freely. An agent can use a function tool, an MCP-provided tool, and a connector-based tool all at the same time — it doesn't notice the difference.

## What's Next

- [What is AI Agent Memory?](what-is-agent-memory.md) — How an agent remembers context across turns.
- [What is MCP?](what-is-mcp.md) — A protocol for sharing tools between agents and AI applications.
- [Adding Tools to an Agent](/docs/genai/develop/agents/adding-tools) — Build tools in WSO2 Integrator.
