---
sidebar_position: 5
title: What is AI Agent Memory?
description: A plain-English explanation of agent memory — how an AI agent remembers what was said earlier in a conversation.
---

# What is AI Agent Memory?

**Memory** is what lets an [AI agent](what-is-ai-agent.md) remember what was said earlier in the conversation. Without it, every message is treated as the very first one — the agent has amnesia between turns. With it, you can have a real back-and-forth conversation: ask a question, follow up, refine, correct.

## The Problem Memory Solves

Compare these two conversations.

**Without memory:**

> User: *"What's the status of order ORD-1042?"*
> Agent: *"ORD-1042 has shipped and arrives Friday."*
> User: *"And ORD-1043?"*
> Agent: *"I don't have any context for that. What would you like to know about it?"*

**With memory:**

> User: *"What's the status of order ORD-1042?"*
> Agent: *"ORD-1042 has shipped and arrives Friday."*
> User: *"And ORD-1043?"*
> Agent: *"ORD-1043 is still being packed and will ship tomorrow."*

The second agent understands that *"And ORD-1043?"* is a continuation. It knows the user is still asking about order status. The reason it knows is memory: each new turn comes with the previous turns attached.

## How Memory Works, Mechanically

Under the hood, memory is just a list of messages. Each time the user says something:

1. The agent looks up the recent messages for the current conversation.
2. It sends those messages, plus the new question, to the LLM.
3. The LLM's reply is appended to the list.
4. The list is saved so the next turn can read it back.

That's it. There is no special "memory module" inside the LLM — the LLM is stateless. *Memory is just the integration replaying the conversation back at every turn.*

This has one important consequence: every message the agent remembers takes up space inside the LLM's context window. A very long conversation costs more (in tokens and money), runs slower, and can eventually exceed the model's limit. That's why memory implementations usually keep only the **last N messages** or summarise older ones.

## Sessions: Keeping Conversations Apart

Memory only works if the agent knows *which* conversation it is in. If two users talk to the same agent at the same time, you do not want User B to see User A's conversation.

The fix is a **session ID** — a string that uniquely identifies one ongoing conversation. Every message the user sends comes tagged with their session ID, and the agent only remembers what was said in that session.

| Session A (Alice) | Session B (Bob) |
|---|---|
| *"What's my last order?"* | *"How do I reset my password?"* |
| *"When will it arrive?"* | *"I forgot my username too."* |

The agent treats those as two completely separate conversations even though they're hitting the same service. Each session is its own little memory bucket.

## How Long Should Memory Last?

This is the main design choice. Three common options:

| Approach | What it means | Good for |
|---|---|---|
| **No memory** | Every call is fresh. The agent has no idea what came before. | One-shot tasks: classify this email, summarise this document. |
| **Short-term memory** | The agent remembers the last few messages of *this* session. When the program restarts, those messages are gone. | Most chat assistants. Lightweight, fast, no infrastructure. |
| **Persistent memory** | Conversation history is saved to a database, so it survives restarts and can span days. | Long-running customer support, multi-day workflows, audit/compliance. |

Many integrations start with no memory or short-term memory and only move to persistent memory when there is a real reason — typically because conversations span multiple sessions or because compliance requires a permanent record.

## Memory vs. Context vs. Knowledge

These three sound similar but solve different problems:

- **Memory** is *"what we already said in this conversation"*. It's about continuity inside one session.
- **Context** is *"facts I want to make sure the agent knows for this one turn"* — for example, attaching the user's profile to the message. It can be used once and not stored.
- **Knowledge** is *"a bigger pool of information the agent can search when needed"* — typically through [RAG](what-is-rag.md). Knowledge isn't tied to a conversation; it's company-wide and reusable.

A well-designed agent often uses all three: it remembers the conversation (memory), it knows the user's role on each request (context), and it can look up policies or documentation when relevant (knowledge).

## Practical Tips

- **Start without memory** if you can. Add it only when the user actually needs follow-ups.
- **Keep memory short.** A short rolling window of recent messages is usually enough for natural conversation and avoids ballooning costs.
- **Be careful with sensitive data.** Whatever ends up in memory is replayed back to the LLM on every turn — including any personal details the user mentioned. If you must store memory, treat the storage like any other personal-data system.
- **One session per real conversation.** Don't reuse session IDs across different users or different topics, or memory will leak between them.

## What's Next

- [What are Tools?](what-are-tools.md) — How an agent gets things done, not just remembers.
- [What is RAG?](what-is-rag.md) — Giving an agent access to a body of knowledge beyond the conversation.
- [Memory](/docs/genai/develop/agents/memory) — Configure memory in WSO2 Integrator.
