---
title: "Troubleshooting and Common Issues"
description: "Diagnose common agent failure modes including infinite loops, hallucination, and wrong tool selection with practical debugging strategies."
---

# Troubleshooting and Common Issues

Agents fail in ways that traditional integrations do not. They can loop endlessly, hallucinate facts, call the wrong tool, or produce plausible-sounding but incorrect results. Debugging these issues requires specialized techniques that go beyond standard log analysis. This guide covers the most common agent failure modes and gives you practical strategies to identify and fix them.

## Common Failure Modes

### Infinite Loops

An agent enters an infinite loop when it repeatedly calls the same tool or cycles between two actions without making progress.

**Symptoms:**
- Rapidly increasing token consumption
- Repeated identical tool calls in conversation logs
- Agent never returns a final response

**Detection:**

```ballerina
import ballerina/log;

configurable int maxIterations = 15;
configurable int maxConsecutiveSameToolCalls = 3;

type ToolCallRecord record {|
    string toolName;
    string inputHash;
|};

function detectLoop(ToolCallRecord[] history) returns boolean {
    if history.length() > maxIterations {
        log:printError("Agent exceeded maximum iterations");
        return true;
    }

    // Check for consecutive identical tool calls
    if history.length() >= maxConsecutiveSameToolCalls {
        int len = history.length();
        boolean allSame = true;
        foreach int i in (len - maxConsecutiveSameToolCalls) ..< len {
            if history[i].toolName != history[len - 1].toolName ||
                    history[i].inputHash != history[len - 1].inputHash {
                allSame = false;
                break;
            }
        }
        if allSame {
            log:printError(string `Agent loop detected: ${history[len - 1].toolName} called ${maxConsecutiveSameToolCalls} times with same input`);
            return true;
        }
    }

    return false;
}
```

**Fix:** Add iteration limits and duplicate-call detection to your agent loop. Force the agent to respond with its best answer when the limit is reached.

### Hallucination

The agent generates plausible but factually incorrect information, especially when it does not have access to the data it needs.

**Symptoms:**
- Responses contain fabricated data (IDs, dates, numbers)
- Agent answers questions about data it was not given
- Tool results are ignored in favor of generated content

**Detection:**

```ballerina
import ballerinax/ai;

type FactCheckResult record {|
    boolean grounded;
    string[] unsupportedClaims;
|};

final ai:Client factCheckClient = check new ({model: "gpt-4o-mini"});

function checkGrounding(string agentResponse, string[] toolResults) returns FactCheckResult|error {
    string context = string:'join("\n---\n", ...toolResults);

    return factCheckClient->generate(
        string `Compare the response against the provided source data.
                Identify any claims in the response that are NOT supported by the source data.

                Source data:
                ${context}

                Response to check:
                ${agentResponse}`,
        FactCheckResult
    );
}
```

:::warning
Grounding checks add an extra LLM call per request. Use them selectively -- for high-stakes outputs or during a debugging session -- rather than on every request in production.
:::

**Fix:** Improve tool descriptions so the agent knows what data it can access. Add system prompt instructions that explicitly prohibit fabricating data.

### Wrong Tool Selection

The agent chooses an inappropriate tool for the task, leading to errors or irrelevant results.

**Symptoms:**
- Tool returns errors because it received unexpected parameters
- Agent uses a general search when a specific lookup was appropriate
- Multiple tool calls where one would suffice

**Detection:**

```ballerina
type ToolSelectionAudit record {|
    string userQuery;
    string selectedTool;
    string[] availableTools;
    boolean resultUsedInResponse;
|};

function auditToolSelection(ToolSelectionAudit audit) {
    log:printInfo(string `TOOL_AUDIT | query="${audit.userQuery}" | ` +
        string `selected=${audit.selectedTool} | ` +
        string `available=${audit.availableTools.toString()} | ` +
        string `result_used=${audit.resultUsedInResponse}`);

    if !audit.resultUsedInResponse {
        log:printWarn(string `Tool result was not used in response -- possible wrong tool selection`);
    }
}
```

**Fix:** Write clearer tool descriptions with explicit use cases and parameter requirements. Add negative examples ("Do NOT use this tool for...").

## Debugging Strategies

### Strategy 1: Conversation Replay

Reproduce an issue by replaying the exact conversation that caused it:

```ballerina
import ballerina/io;

type ReplayStep record {|
    string role;
    string content;
    string? expectedTool;
|};

function replayConversation(ReplayStep[] steps) returns error? {
    foreach int i in 0 ..< steps.length() {
        ReplayStep step = steps[i];
        io:println(string `--- Turn ${i + 1} (${step.role}) ---`);
        io:println(step.content);

        if step.role == "user" {
            string response = check aiClient->generate(step.content, string);
            io:println(string `Agent response: ${response}`);
        }

        io:println("");
    }
}
```

:::tip
Save conversation logs from production with enough detail to replay them in a test environment. Include the system prompt, model version, and temperature setting to get reproducible results.
:::

### Strategy 2: Step-by-Step Agent Inspection

Add verbose logging that captures the agent's internal reasoning at each step:

```ballerina
configurable boolean debugMode = false;

function agentStepWithDebug(string stepName, string input) returns string|error {
    if debugMode {
        log:printInfo(string `[DEBUG] Step: ${stepName}`);
        log:printInfo(string `[DEBUG] Input: ${input.length() > 500 ? input.substring(0, 500) + "..." : input}`);
    }

    time:Utc startTime = time:utcNow();
    string result = check aiClient->generate(input, string);
    decimal durationMs = <decimal>(time:utcNow()[0] - startTime[0]) * 1000d;

    if debugMode {
        log:printInfo(string `[DEBUG] Output: ${result.length() > 500 ? result.substring(0, 500) + "..." : result}`);
        log:printInfo(string `[DEBUG] Duration: ${durationMs}ms`);
    }

    return result;
}
```

### Strategy 3: A/B Testing Prompts

Compare different prompt versions to find which produces better agent behavior:

```ballerina
type PromptVariant record {|
    string name;
    string systemPrompt;
|};

type TestResult record {|
    string variant;
    boolean success;
    int toolCalls;
    decimal latencyMs;
    string response;
|};

function testPromptVariants(PromptVariant[] variants, string testInput) returns TestResult[]|error {
    TestResult[] results = [];

    foreach PromptVariant variant in variants {
        time:Utc start = time:utcNow();
        string|error response = aiClient->generate(
            variant.systemPrompt + "\n\nUser: " + testInput,
            string
        );
        decimal latency = <decimal>(time:utcNow()[0] - start[0]) * 1000d;

        results.push({
            variant: variant.name,
            success: response is string,
            toolCalls: 0,
            latencyMs: latency,
            response: response is string ? response : response.message()
        });
    }

    return results;
}
```

## Debugging Checklist

When an agent misbehaves, work through this checklist:

1. **Check conversation logs** -- Review the full conversation history for the failing request
2. **Verify tool availability** -- Confirm all expected tools are registered and accessible
3. **Inspect tool descriptions** -- Ensure tool descriptions clearly explain when to use each tool
4. **Review the system prompt** -- Look for ambiguous instructions the agent might misinterpret
5. **Check context window usage** -- Verify the conversation has not exceeded the model's limit
6. **Test with a different model** -- Some models handle specific tasks better than others
7. **Reduce temperature** -- Lower temperature for more deterministic behavior
8. **Enable tracing** -- Use OpenTelemetry traces to see the full execution path
