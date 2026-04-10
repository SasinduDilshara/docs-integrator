---
sidebar_position: 2
title: "Build a Smart Calculator Assistant"
description: Build your first AI integration — a calculator agent that uses tools to perform math operations accurately.
---

# Build a Smart Calculator Assistant

**Time:** 10 minutes | **What you'll build:** A chat agent that understands natural-language math requests and calls Ballerina tool functions to perform calculations.

This walkthrough demonstrates the core agent pattern: the LLM reasons about the user's request and decides which tools to call, while the tools do the actual computation. The LLM never does the math itself, which is what makes the answers reliable.

## Prerequisites

- [WSO2 Integrator set up for AI](setup.md)
- A working model provider (either the default WSO2 provider or your own API key)

## Step 1: Define Calculator Tools

Tools are `isolated` Ballerina functions annotated with `@ai:AgentTool`. The tool name, parameter descriptions, and input schema are generated automatically from the function signature and its Ballerina doc comments — you do not need to specify them manually.

Create a file named `tools.bal`:

```ballerina
import ballerina/ai;

# Adds two numbers.
#
# + a - First number
# + b - Second number
# + return - The sum of `a` and `b`
@ai:AgentTool
isolated function add(float a, float b) returns float => a + b;

# Subtracts the second number from the first.
#
# + a - Number to subtract from
# + b - Number to subtract
# + return - The result of `a - b`
@ai:AgentTool
isolated function subtract(float a, float b) returns float => a - b;

# Multiplies two numbers.
#
# + a - First number
# + b - Second number
# + return - The product of `a` and `b`
@ai:AgentTool
isolated function multiply(float a, float b) returns float => a * b;

# Divides the first number by the second.
#
# + a - Dividend
# + b - Divisor
# + return - The result of `a / b`, or an error if `b` is zero
@ai:AgentTool
isolated function divide(float a, float b) returns float|error {
    if b == 0.0 {
        return error("Cannot divide by zero.");
    }
    return a / b;
}
```

:::tip
The LLM reads the doc comment above each tool to decide when to call it. Write the summary line and parameter descriptions as if you were explaining the tool to another developer.
:::

## Step 2: Create the Agent

Create `main.bal` and initialize an `ai:Agent` with the four tools:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:Agent calculatorAgent = check new (
    systemPrompt = {
        role: "Smart Calculator",
        instructions: string `You are a smart calculator assistant.
            When users ask math questions, use the calculator tools to
            compute the answer. Never do arithmetic in your head — always
            call a tool. Show the steps in your response, and chain tool
            calls for multi-step problems.`
    },
    tools = [add, subtract, multiply, divide],
    model = check ai:getDefaultModelProvider()
);

public function main() returns error? {
    while true {
        string userInput = io:readln("You (or 'exit' to quit): ");
        if userInput == "exit" {
            break;
        }
        string response = check calculatorAgent.run(userInput);
        io:println("Agent: ", response);
    }
}
```

## Step 3: Run and Try It

Run the project:

```bash
bal run
```

Then interact with the agent from the terminal:

```
You (or 'exit' to quit): What is 42 multiplied by 17?
Agent: I used multiply(42, 17) and got 714. So 42 × 17 = 714.

You (or 'exit' to quit): If I have 150 items split into 6 groups, and I add 10 more to each group, how many are in each group?
Agent: I first called divide(150, 6) which gave 25, then add(25, 10) which gave 35. Each group has 35 items.
```

Notice how the second prompt requires **two** tool calls in sequence — the agent chains them automatically.

## (Optional) Expose the Agent as a Chat Service

To expose the agent over HTTP, attach a service to an `ai:Listener`. Session-scoped memory is handled by the agent runtime when you pass the `sessionId`:

```ballerina
import ballerina/ai;

service /calculator on new ai:Listener(8090) {
    resource function post chat(ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        string response = check calculatorAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

Test with curl:

```bash
curl -X POST http://localhost:8090/calculator/chat \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "test-1", "message": "What is 42 multiplied by 17?"}'
```

## How It Works

The agent runs a **Reason → Act → Observe** loop on every user message:

1. **Reason** — The LLM reads the user message and decides whether a tool call is needed.
2. **Act** — The agent runtime invokes the chosen Ballerina function with validated arguments.
3. **Observe** — The return value is fed back into the LLM's context.
4. **Repeat or Respond** — The LLM either calls another tool or produces the final response.

Because every arithmetic operation goes through a Ballerina function, the answers are deterministic and precise — the LLM's job is only to pick the right tool.

## What's Next

- [Build a Sample Hotel Booking Agent](hotel-booking-agent.md) — Add memory and multiple domain tools
- [What is an AI Agent?](/docs/genai/key-concepts/what-is-ai-agent) — Understand the agent architecture
- [What are Tools?](/docs/genai/key-concepts/what-are-tools) — Learn tool design patterns
