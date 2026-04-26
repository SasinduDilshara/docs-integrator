---
sidebar_position: 2
title: Creating a Natural Function
description: Step-by-step walkthrough for creating a natural function in WSO2 Integrator BI — the form, parameters, return type, and the Prompt node.
---

# Creating a Natural Function

This page is a click-by-click walkthrough of creating a natural function inside the BI editor. It assumes you have an integration project open. By the end you will have a typed natural function, a return-type record, and a saved English prompt — all without writing Ballerina source by hand.

If you only need the conceptual idea, see [What is a Natural Function?](/docs/genai/key-concepts/what-is-natural-function). If you already have a function and want to call it from a flow, jump to [Calling from a Flow](calling-from-a-flow.md).

## Where to Start From

There are two equivalent entry points. Use whichever is closer at hand.

### From the Project Sidebar

The project sidebar has a dedicated **Natural Functions** node. Hover it and click the **+** icon that appears on the right.

![Project sidebar with the Natural Functions node hovered, showing the inline + button on the right used to create a new natural function.](/img/genai/develop/natural-functions/10-sidebar-natural-functions.png)

### From the Artifacts Panel

From the integration **Overview** page, click **+ Add Artifact**. Scroll to the **Other Artifacts** group and pick **Natural Function** (badged *Beta*).

![The Add Artifact panel scrolled to the Other Artifacts section, with tiles for Function, Natural Function (Beta), Data Mapper, Type, Connection, Configuration. The cursor is over the Natural Function tile.](/img/genai/develop/natural-functions/11-artifacts-other-natural-function.png)

Both paths open the same form: **Create New Natural Function**.

## The Create Form

The form has three fields and a **Create** button:

![Empty Create New Natural Function form with fields for Name, Parameters (Add Parameter link), and Return Type, plus a Create button.](/img/genai/develop/natural-functions/12-create-form-empty.png)

| Field | What it does |
|---|---|
| **Name** | Identifier for the function. Use camelCase, just like any Ballerina function. The editor pre-fills `naturalFunction`. |
| **Parameters** | Zero or more typed inputs the function accepts. Each parameter is a `{type, name, description}` triple. |
| **Return Type** | The shape of the value the function produces. Drives the JSON schema sent to the LLM. |

Fill **Name**, add at least one parameter, set a return type, then click **Create**.

## Adding a Parameter

Click **+ Add Parameter**. A small inline dialog appears:

![Add Parameter dialog with Type field set to string, Name field set to customerReview, and a Description field reading "Review of the customer". Cancel and Save buttons at the bottom right.](/img/genai/develop/natural-functions/13-add-parameter-dialog.png)

| Field | Notes |
|---|---|
| **Type** | A primitive (`string`, `int`, `float`, `decimal`, `boolean`), an array, or any record visible in the project's Type Browser. |
| **Name** | Parameter name in camelCase. This is the name you will reference inside the prompt with `${...}`. |
| **Description** | Short, plain-English description of the parameter. Surfaced to the LLM as part of the schema, and shown in BI when callers bind arguments. |

Click **Save**. The parameter appears as a pill in the Parameters list with edit and delete icons. Repeat to add more.

## Choosing a Return Type

Click the **Return Type** field. A picker drops down with primitive types, plus a **Create New Type** option and an **Open Type Browser** link.

![Return Type dropdown opened, listing Primitive Types (string, int, float, decimal, boolean), with a Create New Type entry and an Open Type Browser link at the bottom.](/img/genai/develop/natural-functions/14-return-type-dropdown.png)

For one-shot text outputs a `string` is fine. For most natural functions you will want a **record** so each output field is named and typed independently.

### Create New Type — From Scratch

Pick **Create New Type**. The **Create New Type** dialog opens on the **Create from scratch** tab.

![Create New Type dialog on the Create from scratch tab, with Kind set to Record, Name field reading ReviewResponse, an empty Fields section with a + button, and Advanced Options collapsed.](/img/genai/develop/natural-functions/15-create-type-scratch.png)

| Field | Notes |
|---|---|
| **Kind** | `Record` is the common choice. The picker also supports `Enum`, `Union`, etc. |
| **Name** | The type identifier (e.g. `ReviewResponse`). |
| **Fields** | Click **+** to add each field with name, type, and an optional description. The description is the line the LLM sees in the schema. |
| **Advanced Options** | Closed/open record toggle, default values, and other type-level switches. |

### Create New Type — Import JSON

If you already have a sample of the JSON shape you want, switch to the **Import** tab. BI generates the record type for you.

![Create New Type dialog on the Import tab, with Format set to JSON, Name set to ReviewResponse, an Import JSON File button, and a textarea showing a pasted JSON sample with sentiment, summary, topics, churn_risk, and suggested_action fields. The Import button is at the bottom right.](/img/genai/develop/natural-functions/16-create-type-import-json.png)

| Field | Notes |
|---|---|
| **Format** | `JSON` (XML and others are also supported). |
| **Name** | The top-level type name to generate. |
| **Import JSON File** | Loads a `.json` file from disk. Or paste directly into the textarea below. |

Click **Import**. BI walks the JSON, infers field types (including nested records and arrays), and adds them to the project under **Types**. For example, the sample above produces:

- `ReviewResponse` with `sentiment`, `summary`, `topics[]`, `churn_risk`, `suggested_action`.
- `Topics` (the array element record) with `name` and `sentiment`.

The new type is selected automatically as the function's return type.

### Picking an Existing Type

If the type already exists in the project, just type its name in the Return Type box and pick from the autocomplete list, or click **Open Type Browser** to browse all visible types in the workspace.

## Saving the Signature

With **Name**, at least one **Parameter**, and a **Return Type** set, click **Create**.

![Create New Natural Function form filled in: Name analyzeCustomerReviewes, one parameter pill string customerReview, Return Type ReviewResponse, Create button enabled.](/img/genai/develop/natural-functions/17-create-form-filled.png)

BI generates the function source and opens it in the **Flow Designer**. The sidebar tree updates in three places:

- **Connections** gets a `_<functionName>Model` entry — the model-provider connection used by this function.
- **Types** lists every record the function depends on (here `ReviewResponse`, `Topics`, `TopicsItem`).
- **Natural Functions** lists the function itself.

## The Natural Function Flow

Once created, the natural function opens with a single **Prompt** node between **Start** and the end of the flow.

![Natural function flow showing Start, a single Prompt node with placeholder text "Enter your prompt here..." and a small cog icon to its right, then the end marker.](/img/genai/develop/natural-functions/18-natural-function-flow-empty.png)

This is the visual representation of the underlying [`natural { }` expression](the-natural-block.md). The Prompt node has two controls you will use next:

- **Pencil icon** — edit the prompt body.
- **Cog icon (right of the node)** — configure which model provider runs this function.

## Configure the Model Provider

Hover the cog icon on the right of the Prompt node. The tooltip reads **Configure Model Provider**.

![Prompt node with the cog icon highlighted on the right and a tooltip reading "Configure Model Provider".](/img/genai/develop/natural-functions/19-prompt-cog-tooltip.png)

Click it. The **Configure Model Provider Connection** panel slides in.

![Configure Model Provider Connection panel showing a Select Model Provider dropdown set to _analyzeCustomerReviewesModel, a + Create New Model Provider link, a hint about the Ballerina: Configure default WSO2 model provider command, and a Save button.](/img/genai/develop/natural-functions/20-configure-model-provider-panel.png)

Three options:

| Option | Use it when |
|---|---|
| **Select Model Provider** dropdown | A provider already exists in the project — pick it and click **Save**. |
| **+ Create New Model Provider** | You need a new provider (OpenAI, Anthropic, Azure OpenAI, …). |
| The **Ballerina: Configure default WSO2 model provider** command palette command | First-time setup of the WSO2-hosted default provider. |

Clicking **+ Create New Model Provider** opens the standard provider catalogue:

![Model Providers picker listing Default Model Provider (WSO2), Anthropic, Azure OpenAI, Deepseek, Google Vertex, Mistral, Ollama, OpenAI, with a tooltip on the WSO2 provider explaining it offers chat completion via WSO2's AI service.](/img/genai/develop/natural-functions/21-model-providers-list.png)

Pick a provider, fill its form (API key, deployment name, etc.), click **Save**. The Prompt node is now bound to that provider.

:::tip One-time WSO2 default
For the WSO2-hosted default provider, run **Ballerina: Configure default WSO2 model provider** from the Command Palette once per project. Sign in with your WSO2 account when prompted; BI persists the configuration. The Configure Model Provider panel will then show it as the default selection.
:::

## Write the Prompt

Click the pencil icon on the Prompt node.

![Prompt node with the pencil icon highlighted at the top right and a tooltip reading "Edit Prompt".](/img/genai/develop/natural-functions/22-prompt-edit-pencil.png)

A small inline editor opens with **Cancel**, **Save**, and an **Expand Editor** icon. Click the expand icon for a full editor with formatting tools.

![Expanded Prompt editor with a toolbar containing Insert, Undo/Redo, Bold, Italic, Underline, Link, H1 dropdown, blockquote, lists, table, and clear-formatting buttons, plus Preview/Source toggle on the right.](/img/genai/develop/natural-functions/23-prompt-rich-editor-empty.png)

| Toolbar item | What it does |
|---|---|
| **Insert** | Insert an interpolation `${parameter}` for any in-scope variable, or a snippet (e.g., a code block). |
| **Bold / Italic / Underline** | Inline emphasis. The model treats Markdown-style emphasis as a hint that the text is important. |
| **H1**, **lists**, **blockquote**, **table** | Structural Markdown. Useful for *Rules:*-style bullets and *Examples:*-style sections. |
| **Preview / Source** | Toggle between rendered Markdown and the raw text the runtime will send. |

Type the task description, mark up key terms, and click anywhere outside to keep the rich view, or **Source** to see the literal prompt:

![Expanded Prompt editor with the prompt typed: "You are a customer review analyzer. For each review, identify the overall sentiment, extract the key topics being discussed with their individual sentiment, and suggest a follow-up action if needed." The phrase customer review analyzer is bold.](/img/genai/develop/natural-functions/24-prompt-rich-editor-filled.png)

Use `${customerReview}`, `${reviews}`, and any other expression to pull parameters into the prompt. See the [Insert menu in The natural { } Block](the-natural-block.md#interpolation) for the full syntax.

Click outside the dialog (or the minimise icon) to collapse it back to the inline view, then **Save**.

![Prompt node showing the saved prompt body inline, with the pencil icon visible for re-editing.](/img/genai/develop/natural-functions/25-prompt-saved.png)

## What BI Generated

Behind the form, BI produced standard Ballerina source. Open the function in **Source View** to read it; for the example above it looks like:

```ballerina
function analyzeCustomerReviewes(string customerReview) returns ReviewResponse|error {
    ReviewResponse|error result = natural {
        You are a **customer review analyzer**. For each review, identify the
        overall sentiment, extract the key topics being discussed with their
        individual sentiment, and suggest a follow-up action if needed.

        Review: ${customerReview}
    };
    return result;
}
```

The form fields map one-to-one onto the source: parameter list → typed signature, return type → return type, prompt body → `natural { ... }` block, model provider → connection used at runtime.

## What's Next

- **[Calling from a Flow](calling-from-a-flow.md)** — call the function you just created from an HTTP resource, automation, or another function.
- **[The `natural { }` block](the-natural-block.md)** — what to put (and *not* put) inside the prompt.
- **[Typed Return Inference](typed-return-inference.md)** — get the most out of the return type.
- **[Review Summarizer with Natural Function](/docs/genai/tutorials/review-summarizer-natural-function)** — an end-to-end tutorial using the same flow.
