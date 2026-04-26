---
sidebar_position: 9
title: Typed Responses
description: How WSO2 Integrator binds LLM responses to Ballerina types automatically — pick the type, leave the schema out of the prompt, and the next node receives a real typed value.
---

# Typed Responses

WSO2 Integrator turns an LLM response into a **typed Ballerina value automatically**. You declare the shape you want — a `string`, a record, an array of records, a union — and the runtime handles the rest:

1. It derives a JSON schema from the type.
2. It instructs the LLM to fill the schema.
3. It parses the response back into a typed value.
4. It hands that value to the next node in your flow.

The next node receives a real `string`, `int`, record, or array — not a blob of text you have to parse yourself.

This applies wherever a model is invoked with an Expected Type:

- The **Expected Type** field on a `generate` node ([Direct LLM Calls](/docs/genai/develop/direct-llm/overview)).
- The return type of a [Natural Function](/docs/genai/develop/natural-functions/overview).
- The result type of an [Agent](/docs/genai/develop/agents/overview)'s final answer.

The point of type binding is to keep the result **accurate, user-friendly, and predictable**.

## Pick a Type, Not a Prompt Instruction

You define the shape you want by **picking a type**. For a structured result, that type is usually a small project record. For example, a record called `BlogContent` with a `title` and a `topic` looks like this in the BI type editor:

![A BI type card titled BlogContent with two fields: title (string) and topic (string).](/img/genai/develop/direct-llm/19-blogcontent-type-card.png)

Set **Expected Type** to `BlogContent` on the `generate` node and the result variable lands in the next node already typed: `blogContent.title` and `blogContent.topic` are real string fields.

| Use this Expected Type | When you want… |
|---|---|
| `string` | Free-form text: a summary, an email body, a translation. |
| A scalar (`int`, `decimal`, `boolean`) | A single number or yes/no answer. |
| A record (e.g. `BlogContent`) | Named fields — the most common choice in integrations. |
| An array of records (e.g. `Topic[]`) | A list of items: extracted entities, suggestions. |
| A union (e.g. `Approved\|Rejected`) | The answer is one of several shapes; `match` on it. |

## Don't Put the Schema in the Prompt

Because the type already drives the schema, **writing *"please return JSON with fields title, topic, …"* in the prompt is wrong**. It is redundant when the prompt agrees with the type, and it fails in production when they don't:

- The type is enforced by the runtime; the prompt is only a hint to the model. If they disagree, the type wins and the call may error on outputs the prompt encouraged.
- A change to the type silently invalidates the hand-written schema in the prompt.
- The prompt grows brittle and hard to read, mixing instructions for the user task with structural noise.

**Set the Expected Type, leave the schema out of the prompt.** The prompt should describe the *task*; the type drives the *shape*.

## When the LLM Doesn't Comply

The runtime returns `T|error`. If the model produces something the type can't accept, an error propagates up your flow — wrap with an error handler to retry or fall back.

A few realities worth knowing:

- **Strict (closed) records** are easier to validate than open ones, and produce clearer errors when the model returns extra fields.
- **The Default WSO2 Model Provider** and **modern flagship models** from each provider are reliable on schema adherence in practice.
- **Smaller, simpler types** are followed more reliably than deep, nested ones.

## Tips for Result Types

- **Use plain field names that match how the task is described.** `summary` not `respText`.
- **Use enums (singleton union types) for fixed sets of values.** `"positive"|"negative"|"mixed"` rather than `string` — this gives the model an exact set of allowed answers, and the runtime rejects anything outside the set.
- **Keep types small.** Two or three fields beat fifteen.
- **Avoid nesting deeper than two levels.** It confuses the model and bloats the response.
- **Add field doc comments** (`# what this field is`). They are sent to the model alongside the schema and improve accuracy noticeably.

## Common Symptoms

| Symptom | Likely cause | Fix |
|---|---|---|
| Response comes back as a string of JSON instead of a parsed record. | Expected Type is `string`. | Set Expected Type to the record type. |
| Parsing fails for some inputs in production. | The prompt and the type disagree, or the type is too loose. | Remove schema instructions from the prompt; use closed records. |
| LLM picks a value outside the allowed set. | Used `string` instead of an enum union. | Switch to a singleton union: `"a"\|"b"\|"c"`. |
| Response stops half-way. | Hit the model's max output tokens. | Raise **Maximum Tokens** in the model provider's [Advanced Configurations](/docs/genai/develop/components/model-providers#standard-http-advanced-configurations), or shrink the type. |

## What's Next

- **[Direct LLM Calls](/docs/genai/develop/direct-llm/overview)** — where the Expected Type is set on the `generate` node.
- **[Writing Effective Prompts](/docs/genai/key-concepts/writing-effective-prompts)** — what the prompt should contain (and what it should not).
- **[Natural Functions](/docs/genai/develop/natural-functions/overview)** — typed return inference from the function signature.
