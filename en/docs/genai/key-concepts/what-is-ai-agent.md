---
sidebar_position: 3
title: What is an AI Agent?
description: A plain-English explanation of AI agents — LLMs that can make decisions and take actions on their own, not just answer questions.
---

# What is an AI Agent?

An **AI agent** is what you get when you give an LLM the ability to **take actions** instead of only producing text.

A plain LLM is like a brilliant friend on the phone — they can think and talk, but they can't reach into your computer to actually *do* anything. An AI agent is the same friend with a set of buttons in front of them: a button to look up a customer, a button to create a ticket, a button to send an email. Now they can decide *and* act.

If you have used something like a chatbot that can actually book a flight, look up your order, or update your profile, you have used an agent.

## How an Agent Thinks

An agent works in a small loop, sometimes called the **Reason → Act → Observe** loop:

```bash
       ┌──────────── User says something ────────────┐
       │                                              │
       ▼                                              │
   ┌────────┐    decides what to do next              │
   │ Reason │ ───────────────────────────────┐        │
   └────────┘                                ▼        │
                                          ┌─────┐     │
                                          │ Act │  ◄── one of the buttons
                                          └─────┘
                                             │
                                             ▼
                                       ┌─────────┐
                                       │ Observe │ ◄── the result of pressing the button
                                       └─────────┘
                                             │
                                             ▼
                                  enough info to answer?
                                  ┌──── no ────┐
                                  ▼            └──► loop again
                          tell the user
```

In words:

1. **Reason** — The LLM looks at the user's question, what it already knows, and what buttons (tools) are available. It decides whether it can answer right now or whether it needs to press a button first.
2. **Act** — If the LLM decides to use a tool, the agent runtime calls that tool with the arguments the LLM chose. The LLM does *not* run code itself — it just says *"please call this tool with these arguments"*.
3. **Observe** — The result of the tool comes back. The LLM now has new information.
4. **Repeat or finish.** It might press another button, then another. Eventually it has enough to give the user a real answer.

This is what makes an agent feel "smart" — it doesn't just produce text in one shot. It pulls in the data it needs, step by step, until the answer is grounded in the real world.

## What an Agent Is Made Of

Every agent has four ingredients:

| Ingredient | What it does |
|---|---|
| **Model** | The LLM. The brain that does the reasoning. |
| **System prompt** | A short briefing for the agent: *who* it is, *what* it should do, *what* it shouldn't do. |
| **Tools** | The buttons the agent can press — see [What are Tools?](what-are-tools.md). |
| **Memory** | What the agent remembers between turns of a conversation — see [What is AI Agent Memory?](what-is-agent-memory.md). |

The model and system prompt are required. Tools and memory are optional — but an agent without tools is essentially a fancy chatbot, and an agent without memory can't carry on a conversation.

## Agent vs. Natural Function vs. Plain LLM Call

These three look similar from the outside but solve different problems:

| Pattern | What it does | When to use |
|---|---|---|
| **Direct LLM call** | Send one prompt, get one answer back. No structure, no actions. | Quick experiments, simple text generation. |
| **[Natural function](what-is-natural-function.md)** | A typed function whose body is English. Single shot, structured output. | Classification, summarisation, extraction — *one input, one answer*. |
| **AI agent** | An LLM with tools, memory, and the freedom to take multiple steps. | Conversational assistants, anything that has to *do* things. |

A useful rule of thumb: if your task can be answered without ever looking anything up, you don't need an agent. The moment the task becomes *"check this, then maybe that, then respond"* — you do.

## The Difference Between an Agent and a Workflow

It's tempting to ask: "Can't I just write a workflow that calls these APIs in order, and use the LLM only for the wording at the end?"

Often, yes — and you should. A workflow is a great choice when **you know in advance which steps will run, and in what order**. It is fast, predictable, and easy to debug.

An agent is the right answer when **you don't know which steps will run** until the user actually says something. The agent decides the path at runtime, based on the question. That flexibility is what lets one agent handle thousands of slightly different requests, but it also means agent behaviour is harder to predict than a hand-written workflow. Use the right tool for the job.

## What an Agent Cannot Magically Solve

It is easy to assume an agent will figure things out by itself. In practice:

- **An agent only knows what it is told.** If a tool description is vague, the agent will call it badly. If the system prompt is vague, the agent will wander.
- **More tools is not better.** The more tools you give the agent, the harder it has to work to pick the right one. Agents work better with a few well-designed tools than with dozens of overlapping ones.
- **Agents can be wrong.** The LLM is still a probabilistic system. Important actions (refunds, deletions, sending emails) deserve a confirmation step.
- **Agents can be slow and expensive.** Each loop of Reason → Act → Observe is one or more LLM calls. Tasks that take three steps cost roughly three times as much as a single direct LLM call.

> **AI Agent vs. AI Chat Agent in BI.** *AI agent* is the general concept above. In WSO2 Integrator BI, the ready-made artifact you create from **+ Add Artifact → AI Integration → AI Chat Agent** packages a chat-style agent (listener + agent node + return) so you don't have to assemble it yourself. The lower-level **Agent** node under **AI → Agent** in the Add Node panel gives you the same capability inside any flow. Both are the same idea — the artifact is just a faster starting point.

## What's Next

- [What are Tools?](what-are-tools.md) — How an agent learns to take actions.
- [What is AI Agent Memory?](what-is-agent-memory.md) — How an agent remembers a conversation.
- [Creating an AI Agent](/docs/genai/develop/agents/creating-an-agent) — Build one in WSO2 Integrator.
