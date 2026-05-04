---
sidebar_position: 1
title: Natural Functions
description: Single-page reference for Natural Functions in WSO2 Integrator BI — typed Ballerina functions whose body is plain English, evaluated by an LLM at runtime.
---

# Natural Functions

A **natural function** is a Ballerina function whose body is written in **English** instead of code. You declare the signature — typed parameters and a typed return — and you describe the work in a `natural { ... }` block (the **Prompt** node in the BI editor). At runtime, WSO2 Integrator turns the return type into a JSON schema, sends both the prompt and the schema to a model provider, and gives you back a typed value.

This page is a single, end-to-end reference: the BI form that creates the function, the Prompt node, the return type, and how to call it from a flow.

> **Looking for a hands-on walkthrough?** See the **[Customer Review Analyzer with Natural Function](/docs/genai/tutorials/review-summarizer-natural-function)** tutorial — it builds the example shown on this page from an empty project to a working `POST /api/v1/analyze` endpoint.

:::caution Experimental Feature
Natural expressions are an **experimental** language feature. The integration runtime adds the `--experimental` flag automatically when you click **Run** in BI; from the terminal use `bal run --experimental`.
:::

## When to Use a Natural Function

| Use a natural function when… | Look elsewhere when… |
|---|---|
| You have a single-step text task: classify, summarise, extract, rewrite. | The task needs tool calls or multi-step reasoning — use an [AI Agent](/docs/genai/develop/agents/overview). |
| You want the same prompt usable from many flows, fully typed. | You only need a one-off LLM call inside one flow — use a [Direct LLM Call](/docs/genai/develop/direct-llm/overview). |
| You want a function you can mock and unit-test. | The task needs to remember earlier turns of a conversation — use an agent with memory. |
| You can describe the work in two or three sentences. | The work needs your own data — combine with [RAG](/docs/genai/develop/rag/overview). |

## How a Natural Function Looks

A finished natural function is a single **Prompt** node sitting between **Start** and the end of the flow. The cog on the right binds a model provider; the pencil at the top edits the English body.

![Natural function flow with Start, an empty Prompt node ("Enter your prompt here..."), and the end marker. A small cog icon sits to the right of the Prompt node.](/img/genai/develop/natural-functions/18-natural-function-flow-empty.png)

Once the prompt is written and saved, the body shows inline on the node:

![Prompt node with the saved prompt body shown inline, and the pencil icon visible for re-editing.](/img/genai/develop/natural-functions/25-prompt-saved.png)

What you do in the BI editor:

- Declare the function's **signature** (name, parameters, and return type) once in the **Create New Natural Function** form.
- Bind a **Model Provider** to the Prompt node.
- Write the **English instructions** in the Prompt node, with `${...}` interpolations for the parameters.

What WSO2 Integrator handles for you:

- Generating a JSON schema from the function's return type.
- Sending the prompt and the schema to the bound model provider.
- Parsing the response back into a typed return value.
- Surfacing any parsing errors as a Ballerina `error`.

To build one, you do four things in order:

1. [**Create the function**](#creating-a-natural-function) — name, parameters, and return type.
2. [**Bind a model provider**](#configuring-the-model-provider) — once per project, then reused.
3. [**Write the prompt**](#writing-the-prompt) inside the **Prompt** node.
4. [**Call it from a flow**](#calling-from-a-flow).

The rest of this page walks each step.

---

## Creating a Natural Function

There are two equivalent entry points; both open the same form.

**From the project sidebar.** Hover the **Natural Functions** node and click the **+** that appears.

![Project sidebar with the Natural Functions node hovered, showing the inline + button on the right.](/img/genai/develop/natural-functions/10-sidebar-natural-functions.png)

**From the Artifacts panel.** Click **+ Add Artifact** on the integration overview, then under **Other Artifacts** pick **Natural Function** (badged *Beta*).

![Add Artifact panel scrolled to Other Artifacts, with the Natural Function (Beta) tile highlighted.](/img/genai/develop/natural-functions/11-artifacts-other-natural-function.png)

### The Create Form

The form has three fields and a **Create** button.

![Empty Create New Natural Function form with Name, Parameters (Add Parameter link), Return Type, and Create button.](/img/genai/develop/natural-functions/12-create-form-empty.png)

| Field | What it does |
|---|---|
| **Name** | Function identifier, in camelCase. |
| **Parameters** | Zero or more typed inputs. Each parameter is a `{type, name, description}` triple. The name is what you reference inside the prompt with `${...}`. |
| **Return Type** | The shape of the value the function produces. Drives the JSON schema sent to the LLM. |

### Adding a Parameter

Click **+ Add Parameter**. The inline dialog asks for a type, a name, and a short description that's surfaced both to the LLM (as part of the schema) and to callers in BI when they bind arguments.

![Add Parameter dialog with Type string, Name customerReview, Description "Review of the customer", Cancel and Save buttons.](/img/genai/develop/natural-functions/13-add-parameter-dialog.png)

### Choosing a Return Type

Click **Return Type**. The picker offers primitives, plus **Create New Type** and **Open Type Browser**.

![Return Type dropdown listing primitive types with a Create New Type entry and an Open Type Browser link.](/img/genai/develop/natural-functions/14-return-type-dropdown.png)

For most natural functions you want a **record** so each output field is named and typed. The **Create New Type** dialog has two tabs:

- **Create from scratch** — pick `Record`, name the type, add fields with types and descriptions.

  ![Create New Type dialog on the Create from scratch tab, with Kind set to Record, Name field reading ReviewResponse, an empty Fields section with a + button, and Advanced Options collapsed.](/img/genai/develop/natural-functions/15-create-type-scratch.png)

- **Import** — paste a JSON sample (or load a `.json` file) and BI infers the record type, including nested records and arrays. Fastest path when you already have an example response.

  ![Create New Type dialog on the Import tab with Format JSON, Name ReviewResponse, an Import JSON File button, and a JSON sample pasted into the textarea showing sentiment, summary, topics, churn_risk, and suggested_action.](/img/genai/develop/natural-functions/16-create-type-import-json.png)

The new type is selected automatically as the function's return type. With at least a name, one parameter, and a return type, click **Create**.

![Create New Natural Function form filled in: Name analyzeCustomerReviews, parameter pill "string customerReview", Return Type ReviewResponse, Create button enabled.](/img/genai/develop/natural-functions/17-create-form-filled.png)

BI generates the source and opens the function in the **Flow Designer** with three additions in the sidebar: a `_<functionName>Model` connection, the type(s) under **Types**, and the function under **Natural Functions**. The flow itself is just a single **Prompt** node between **Start** and the end.

![Natural function flow with Start, an empty Prompt node ("Enter your prompt here..."), and the end marker. A small cog icon sits to the right of the Prompt node.](/img/genai/develop/natural-functions/18-natural-function-flow-empty.png)

The Prompt node has two controls: the **pencil** (top-right) edits the prompt body, and the **cog** (right of the node) configures the model provider.

---

## Configuring the Model Provider

The Prompt node needs a **Model Provider** connection — the LLM it talks to. Hover the cog on the right of the Prompt node and click **Configure Model Provider** to bind one.

![Prompt node with the cog icon highlighted and a tooltip reading "Configure Model Provider".](/img/genai/develop/natural-functions/19-prompt-cog-tooltip.png)

If a provider already exists in the project, pick it from the dropdown and click **Save**. If not, click **+ Create New Model Provider** and pick from the catalogue (OpenAI, Anthropic, Azure OpenAI, Default WSO2, …).

Adding a provider, the per-provider form fields, the supported models, and the advanced HTTP knobs are all documented in **[AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers)**. You only need to do it once per project — every natural function, direct LLM call, RAG `generate` node, and AI Agent in the project shares the same provider connections.

> The fastest one to set up is the **Default WSO2 Model Provider** — no API key, just a one-time WSO2 sign-in via the Command Palette command **Ballerina: Configure default WSO2 model provider**.

---

## Writing the Prompt

Click the pencil icon at the top-right of the Prompt node. An inline editor opens; click **Expand Editor** for the full Markdown editor with formatting tools.

![Prompt node with the pencil edit icon highlighted and a tooltip reading "Edit Prompt".](/img/genai/develop/natural-functions/22-prompt-edit-pencil.png)

![Expanded Prompt editor showing a toolbar with Insert, Undo/Redo, Bold, Italic, Underline, Link, H1, blockquote, lists, table, clear-formatting, and a Preview/Source toggle.](/img/genai/develop/natural-functions/23-prompt-rich-editor-empty.png)

| Toolbar action | Effect |
|---|---|
| **Insert** | Insert an `${parameter}` interpolation for any in-scope variable, or a snippet (block quote, code block). |
| **Bold / Italic / Underline** | Inline emphasis. The model treats Markdown emphasis as a hint to weight that term. |
| **H1**, **lists**, **blockquote**, **table** | Structural Markdown — useful for *Rules:*-style lists and *Examples:*-style sections. |
| **Preview / Source** | Toggle between rendered Markdown and the literal text the runtime sends. |

A typical filled body:

![Expanded Prompt editor with the prompt typed and the phrase "customer review analyzer" bold.](/img/genai/develop/natural-functions/24-prompt-rich-editor-filled.png)

Click **Save**. The body collapses back into the Prompt node.

![Prompt node showing the saved prompt body inline.](/img/genai/develop/natural-functions/25-prompt-saved.png)

### Interpolation

Any in-scope value can be embedded with `${...}`. Use the **Insert** menu or just type it.

| What you interpolate | What the model sees |
|---|---|
| `${reviewText}` | The string itself, inline. |
| `${reviews}` (an array) | Each item as JSON, joined into a list. |
| `${customer}` (a record) | The record serialised as JSON. |
| `${customer.name}` | Just the field. |
| `${reviews.length()}` | The result of any in-scope expression. |

### Anatomy of a Good Prompt

Three things tend to make natural-function bodies more reliable:

1. **State the task in one sentence at the top.** *"Classify the following customer review as positive, negative, or neutral and explain why."* — not *"Take a look at this and tell me what you think."*
2. **Add rules as a short bullet list.** Models follow bullets better than long paragraphs.
3. **Put the inputs at the end, after a clear divider.** Inputs are easier to spot when they're separated from instructions.

### What NOT to Put in the Prompt

- **The output schema.** The return type drives that. Asking the LLM to *"return JSON with fields …"* is at best redundant and at worst fights the schema.
- **"Return JSON" instructions.** Same reason.
- **Megabytes of unrelated context.** Prompts are paid per token. Use [RAG](/docs/genai/develop/rag/overview) to bring in only what's relevant.
- **Secrets.** Anything in a prompt is sent to the LLM provider on every call.

---

## Typed Return Inference

The headline feature of natural functions is this: **the return type is the contract**. When the runtime encounters a `natural { ... }` block, it:

1. Looks at the declared return type.
2. Builds a JSON schema from that type and includes it in the LLM call.
3. Parses the model's response back into a value of that type.

If the model produces something that doesn't match the type, the runtime asks it to retry once with a clearer schema reminder; if parsing still fails, it returns an `error`. By the time a successful call returns, every required field is present and every value has the right shape — no *"the LLM forgot a field"* failure mode further down the flow.

### Picking a Type

| Return type | When to use it |
|---|---|
| `string` | Free-form text. |
| `int`, `decimal`, `boolean` | A scalar parsed from the model's reply. |
| Record (closed `record \{\| ... \|\}`) | Named, typed fields. The most common choice. |
| Array (`Topic[]`) | A list. |
| Union (`"a"\|"b"\|"c"` or `Approved\|Rejected`) | The variant the model picked, parsed into the right shape. |

### Tips for Result Types

- **Use closed records (`record \{\| ... \|\}`)** when you want exactly these fields. The schema is tighter, the parser is stricter.
- **Use enum unions for fixed-set values** — `"positive"|"negative"|"neutral"`. The model is constrained to pick one.
- **Add field descriptions.** Each field in the type editor takes a short description; that description is included in the JSON schema sent to the LLM. Visualised in the Types editor, a record with a nested array (`Topics`) and a row record (`TopicsItem`) looks like this:

    ![Types editor showing the ReviewResponse record with fields sentiment (string), summary (string), topics (Topics), churn_risk (boolean), and suggested_action (string), connected to a Topics record (containing TopicsItem) and a TopicsItem record with fields name (string) and sentiment (string).](/img/genai/develop/natural-functions/26-review-response-type-diagram.png)

- **Keep nesting shallow.** One level of records and arrays is fine; two starts to confuse smaller models; three usually benefits from being split.

### Don't Repeat the Schema in the Prompt

Because the type already drives the schema, **writing *"please return JSON with fields …"* in the prompt is wrong**. It's redundant when it agrees with the type, and brittle when it doesn't. Set the return type, leave the schema out of the prompt — the prompt describes the *task*; the type drives the *shape*.

---

## Calling from a Flow

Once the function exists, calling it from a flow is one step.

1. Open a resource, automation, or function flow in the **Flow Designer**.
2. Click **+** between two nodes.
3. In the **Add Node** panel, expand the **AI** category and click **Call Natural Function**.

![Add Node panel with the AI category expanded, showing Direct LLM (Model Provider, Call Natural Function) and RAG nodes. The Call Natural Function tile is highlighted.](/img/genai/develop/natural-functions/32-add-node-call-natural-function.png)

4. The **Natural Functions** picker lists every natural function in the current integration. Pick one.

![Natural Functions picker showing a Search box and a Current Integration section with the analyzeCustomerReviews natural function listed.](/img/genai/develop/natural-functions/33-natural-functions-picker.png)

5. The configuration form opens. Each parameter on the function becomes a row; bind it to an in-scope value. **Result** is the variable name that holds the typed return; **Variable Type** is locked to the function's declared return type.

![Configuration form for the analyzeCustomerReviews call: CustomerReview field with Text/Expression toggle, Result name reviewResponse, Variable Type ReviewResponse (locked), Save button.](/img/genai/develop/natural-functions/34-call-config-empty.png)

![Configuration form filled in: CustomerReview bound to the variable review (shown as a pill), Result reviewResponse, Variable Type ReviewResponse, Save button enabled.](/img/genai/develop/natural-functions/35-call-config-filled.png)

6. Click **Save**. A new node appears in the flow, named after the bound result variable. Use that variable like any other typed value: return it from an HTTP resource (it becomes the JSON response), branch on a field with `Match`, transform it with `Map Data`, or pass it to another node.

![Final resource flow: Start → analyzeCustomerReviews (reviewResponse) → Return reviewResponse → Error Handler.](/img/genai/develop/natural-functions/38-final-resource-flow.png)

### Calling from Another Function or Agent

Natural functions are first-class functions, so they can call each other from inside any flow — open a function flow, **+ Add Node → Statement → Call Function**, and pick the natural function from the picker. Two-step prompts (extract first, then transform) are often cleaner than one giant prompt; each step has its own type and can be tested in isolation.

A natural function can also be wired up as an [agent tool](/docs/genai/develop/agents/tools): the agent decides *whether* to call it, and the natural function's own model handles the call with a tighter prompt and stricter return type.

---

## Common Mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| `bal run` fails with *"natural expressions require --experimental"*. | The `--experimental` flag isn't on the command line. | Click **Run** in BI (it adds the flag), or run `bal run --experimental` in the terminal. |
| Compile error: *"Default model for natural functions not configured"*. | First-time setup missed. | Run **Ballerina: Configure default WSO2 model provider** from the Command Palette, or pick a provider via the cog icon on the Prompt node. |
| Returned record has empty / wrong fields. | Prompt doesn't mention the fields the type expects, or descriptions are missing. | Add field doc comments on the type, or spell the fields out by name in the prompt. |
| Function works in isolation but errors when called from a flow. | Bound parameter has a different shape than the function expects. | Check the binding in the Call Natural Function panel; bind to a value with the matching shape. |
| Same input gives different answers each run. | Default temperature isn't 0. | Lower the model provider's temperature in **Connections → Advanced Configurations**. |
| Result type is a union and parsing fails for one variant. | Schema wasn't tight enough — model produced something close but not exactly matching. | Use closed records and singleton-string unions (`"a"\|"b"\|"c"`); add field doc comments. |

## What's Next

- **[Customer Review Analyzer with Natural Function (Tutorial)](/docs/genai/tutorials/review-summarizer-natural-function)** — end-to-end tutorial that builds a `POST /api/v1/analyze` service using everything on this page.
- **[AI Connections and Stores → Model Providers](/docs/genai/develop/components/model-providers)** — switch providers, tune temperature, max tokens, and retries for the Prompt node's connection.
- **[Direct LLM Calls](/docs/genai/develop/direct-llm/overview)** — when you only need a single in-flow call without packaging it as a function.
- **[AI Agents](/docs/genai/develop/agents/overview)** — when natural functions become tools an agent can choose to call.
- **[What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function)** — conceptual background.
