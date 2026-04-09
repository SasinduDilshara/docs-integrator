---
title: "What Is Memory?"
description: "Understand how memory gives AI agents the ability to maintain context across interactions."
---

# What Is Memory?

In the context of AI agents, **memory** is the mechanism that determines what context the agent has access to when reasoning about a user's message. Without memory, every interaction starts from scratch -- the agent has no knowledge of previous messages, preferences, or decisions. With memory, the agent can maintain coherent conversations, recall relevant details, and build on prior interactions.

## Why Memory Matters

LLMs are stateless by nature. Each call to the model is independent -- the LLM does not remember what you asked five minutes ago unless that information is explicitly included in the current request. Memory bridges this gap by managing which previous messages, facts, or interactions are sent to the LLM alongside each new user message.

The right memory configuration can make the difference between:

- An agent that forgets your name after five messages
- An agent that recalls relevant details from weeks ago

Memory is essential for:

- **Multi-turn conversations** -- Maintaining context as a dialogue progresses through multiple exchanges.
- **Personalization** -- Remembering user preferences, past requests, and established context.
- **Complex tasks** -- Keeping track of intermediate results and decisions across multiple reasoning steps.
- **Continuity** -- Allowing users to return to a conversation days later and pick up where they left off.

## Types of Memory

AI agents typically use three types of memory, each suited to different scenarios.

### Conversation Memory (Message Window)

The simplest form of memory. It stores the most recent N messages in the current session and discards anything older.

**How it works:** Each new user message and agent response is appended to the history. When the history exceeds the configured maximum, the oldest messages are dropped.

**Best for:** Short, focused conversations like support tickets, quick Q&A sessions, or task-oriented interactions where older context is not important.

**Trade-off:** Messages dropped from the window are gone permanently. If the agent needs to reference something from earlier in a long conversation, it will not have that context.

### Semantic Memory

Semantic memory uses a vector store to save conversation history and retrieves relevant past messages based on their similarity to the current query. Instead of keeping the last N messages, it keeps the most relevant ones.

**How it works:** Every message is converted to a vector embedding and stored. When a new message arrives, the system finds the most semantically similar past messages and includes them in the context, along with a small window of recent messages.

**Best for:** Long-running conversations where specific topics may resurface, research or advisory agents where users revisit earlier questions, and agents that handle multiple topics in a single session.

**Trade-off:** Requires a vector store and an embedding model, which adds infrastructure complexity and latency.

### Persistent Memory

Persistent memory stores conversation history in a database, enabling agents to recall context across separate sessions. When a user returns days or weeks later, the agent can load their history and continue as if no time had passed.

**How it works:** All messages are persisted to a database. When a session resumes, the agent loads the most recent messages from storage. A session identifier maps incoming requests to stored conversations.

**Best for:** Multi-session interactions (users who return over days or weeks), agents that need to remember user preferences, and compliance scenarios that require conversation audit trails.

**Trade-off:** Requires database infrastructure and careful session management.

## Combining Memory Types

Advanced agents can layer multiple memory types. For example, an agent might use conversation memory for immediate context (the last 10 messages) combined with persistent memory for cross-session recall. This gives the agent both short-term awareness and long-term knowledge.

## Memory in WSO2 Integrator

WSO2 Integrator provides configurable memory strategies that you attach to your agents. The memory configuration determines:

- How many recent messages to retain
- Whether to use vector-based semantic retrieval
- Whether to persist conversations across sessions
- How to identify and map user sessions

You choose the memory type based on your use case -- quick support chats need only a small message window, while personal advisors benefit from persistent storage combined with semantic retrieval.

For detailed configuration options and code examples, see the [Adding Memory to an Agent](../build-ai-applications/ai-agents/adding-memory-to-an-agent.md) guide.

## What's Next

- [What Is an AI Agent?](what-is-an-ai-agent.md) -- How agents use memory within the perceive-reason-act loop
- [What Are Tools?](what-are-tools.md) -- How tools work alongside memory to extend agent capabilities
- [What Is RAG?](what-is-rag/index.md) -- A related concept where external knowledge augments LLM responses
