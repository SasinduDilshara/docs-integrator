---
sidebar_position: 6
title: Customer Review Analyzer with Natural Function
description: Step-by-step tutorial — build an HTTP service that uses a natural function to analyze customer reviews and return structured feedback with sentiment, topics, churn risk, and a suggested action.
---

# Customer Review Analyzer with Natural Function

This tutorial walks through building an **HTTP service that uses a natural function to analyze a customer review** and return structured feedback. It is the end-to-end scenario for the [Natural Functions](/docs/genai/develop/natural-functions/overview) feature.

By the end you will have a `POST /api/v1/analyze` endpoint that takes a single customer review and returns a typed `ReviewResponse` containing the overall sentiment, a concise summary, per-topic sentiment, a churn-risk flag, and a suggested follow-up action — all produced by an LLM, using a natural function as the body.

:::caution Experimental Feature
Natural expressions are experimental. BI applies the `--experimental` flag automatically when you click **Run** in the editor. From the terminal, use `bal run --experimental`.
:::

:::info Default Model Provider
Before you start, open the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) and run **Ballerina: Configure default WSO2 model provider**. Sign in with your WSO2 account when prompted. This is a one-time per-project setup; BI writes the configuration values into the project automatically.

![Command Palette filtered to "Configure default model provider", showing matches "Ballerina: Configure default WSO2 Model Provider" and "Ballerina: Configure default model for natural functions (Experimental)".](/img/genai/develop/natural-functions/39-command-palette-natural-functions.png)

The first run may also prompt you for the `wso2aiKey` configuration value. The **Configure Application** dialog handles that for you.
:::

## What You'll Build

1. **Create the natural function** — typed signature, return type with a record, an English prompt body, and a model-provider connection.
2. **Create the HTTP service** that exposes the function over `POST /api/v1/analyze`.
3. **Wire the resource flow** to call the natural function and return the typed response.
4. **Run and test** the service end to end.

---

## 1. Create the Natural Function

### Step 1.1 — Open the Create Form

From the project sidebar, hover the **Natural Functions** node and click the **+** that appears on the right.

![Project sidebar with the Natural Functions node hovered, showing the inline + button on the right.](/img/genai/develop/natural-functions/10-sidebar-natural-functions.png)

(You can also reach the same form from the integration **Overview** page: click **+ Add Artifact** and pick **Natural Function** under **Other Artifacts**.)

![Add Artifact panel scrolled to Other Artifacts, with the Natural Function (Beta) tile highlighted.](/img/genai/develop/natural-functions/11-artifacts-other-natural-function.png)

### Step 1.2 — Name and Parameter

The **Create New Natural Function** form opens. Set:

| Field | Value |
|---|---|
| **Name** | `analyzeCustomerReviewes` |

![Empty Create New Natural Function form with Name, Parameters (Add Parameter link), Return Type, and Create button.](/img/genai/develop/natural-functions/12-create-form-empty.png)

Click **+ Add Parameter** and fill in:

| Field | Value |
|---|---|
| **Type** | `string` |
| **Name** | `customerReview` |
| **Description** | `Review of the customer` |

![Add Parameter dialog with Type string, Name customerReview, Description "Review of the customer", Cancel and Save buttons.](/img/genai/develop/natural-functions/13-add-parameter-dialog.png)

Click **Save**. The parameter appears as a pill in the Parameters list.

### Step 1.3 — Build the Return Type

The function will return a `ReviewResponse` record. The fastest way to define one is to import a JSON sample.

Click the **Return Type** field. From the dropdown, click **Create New Type**.

![Return Type dropdown listing primitive types with a Create New Type entry and an Open Type Browser link.](/img/genai/develop/natural-functions/14-return-type-dropdown.png)

In the **Create New Type** dialog, switch to the **Import** tab. Set:

| Field | Value |
|---|---|
| **Format** | `JSON` |
| **Name** | `ReviewResponse` |

Paste the following JSON sample into the textarea:

```json
{
  "sentiment": "mixed",
  "summary": "Customer loves the sound quality but is frustrated by a faulty charging case and slow support response.",
  "topics": [
    { "name": "sound quality",     "sentiment": "positive" },
    { "name": "charging case",     "sentiment": "negative" },
    { "name": "customer support",  "sentiment": "negative" }
  ],
  "churn_risk": true,
  "suggested_action": "Reach out with a replacement case and apologize for the support delay."
}
```

![Create New Type dialog on the Import tab with Format JSON, Name ReviewResponse, an Import JSON File button, and the JSON sample pasted into the textarea.](/img/genai/develop/natural-functions/16-create-type-import-json.png)

Click **Import**. BI infers and registers three types under the **Types** node in the sidebar:

- `ReviewResponse` (`sentiment`, `summary`, `topics`, `churn_risk`, `suggested_action`).
- `Topics` (the array element record: `name`, `sentiment`).
- `TopicsItem` (the per-element type for the inferred array).

`ReviewResponse` is selected automatically as the function's return type.

> Why these field names matter: the runtime turns the record (and its nested types) into a JSON schema and sends it to the LLM. Field names, types, and any descriptions you add become part of the contract the model is constrained to. See [Typed Return Inference](/docs/genai/develop/natural-functions/typed-return-inference) for the details.

### Step 1.4 — Create the Function

The form should now look like this:

![Create New Natural Function form filled in: Name analyzeCustomerReviewes, parameter pill "string customerReview", Return Type ReviewResponse, Create button enabled.](/img/genai/develop/natural-functions/17-create-form-filled.png)

Click **Create**. BI generates the Ballerina source and opens the function in the **Flow Designer**. The sidebar updates with three additions:

- **Connections** — `_analyzeCustomerReviewesModel` (the model-provider connection BI created for this function).
- **Types** — `ReviewResponse`, `Topics`, `TopicsItem`.
- **Natural Functions** — `analyzeCustomerReviewes`.

The flow shows a single **Prompt** node between **Start** and the end of the function:

![Natural function flow with Start, an empty Prompt node ("Enter your prompt here..."), and the end marker. A small cog icon sits to the right of the Prompt node.](/img/genai/develop/natural-functions/18-natural-function-flow-empty.png)

### Step 1.5 — Bind the Model Provider

Hover the cog icon on the right of the Prompt node — the tooltip reads **Configure Model Provider**. Click it.

![Prompt node with the cog icon highlighted on the right and a tooltip reading "Configure Model Provider".](/img/genai/develop/natural-functions/19-prompt-cog-tooltip.png)

The **Configure Model Provider Connection** panel slides in. Pick the auto-created `_analyzeCustomerReviewesModel` connection and click **Save**.

![Configure Model Provider Connection panel with Select Model Provider set to _analyzeCustomerReviewesModel, "+ Create New Model Provider" link, a hint about the Configure default WSO2 model provider command, and a Save button.](/img/genai/develop/natural-functions/20-configure-model-provider-panel.png)

If you'd rather use OpenAI, Anthropic, Azure OpenAI, or any other provider, click **+ Create New Model Provider** and pick from the catalogue:

![Model Providers picker listing Default Model Provider (WSO2), Anthropic, Azure OpenAI, Deepseek, Google Vertex, Mistral, Ollama, OpenAI.](/img/genai/develop/natural-functions/21-model-providers-list.png)

### Step 1.6 — Write the Prompt

Click the pencil icon at the top-right of the Prompt node.

![Prompt node with the pencil icon highlighted at the top right and a tooltip reading "Edit Prompt".](/img/genai/develop/natural-functions/22-prompt-edit-pencil.png)

The inline editor opens. Click **Expand Editor** for the full Markdown editor with formatting tools:

![Expanded Prompt editor showing a toolbar with Insert, Undo/Redo, Bold, Italic, Underline, Link, H1, blockquote, lists, table, clear-formatting, and a Preview/Source toggle.](/img/genai/develop/natural-functions/23-prompt-rich-editor-empty.png)

Type the following prompt (use **Bold** for the role line, and the **Insert** menu — or just type — to interpolate the parameter):

> You are a **customer review analyzer**. For each review, identify the overall sentiment, extract the key topics being discussed with their individual sentiment, and suggest a follow-up action if needed.
>
> Review: `${customerReview}`

![Expanded Prompt editor with the prompt typed and the phrase "customer review analyzer" bold.](/img/genai/develop/natural-functions/24-prompt-rich-editor-filled.png)

Close the expanded view and click **Save**. The Prompt node now shows the body inline:

![Prompt node showing the saved prompt body inline.](/img/genai/develop/natural-functions/25-prompt-saved.png)

The natural function is complete. Behind the scenes, BI generated:

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

---

## 2. Create the HTTP Service

The function is callable; now we expose it as an API.

### Step 2.1 — Add the Service Artifact

Click the back arrow to return to the integration **Overview**, then **+ Add Artifact**. Under **Integration as API**, pick **HTTP Service**.

![Add Artifact panel showing Automation, AI Integration, Integration as API (HTTP Service highlighted, GraphQL Service Beta, TCP Service Beta), Event Integration, and File Integration.](/img/genai/develop/natural-functions/26-artifacts-http-service.png)

In the **Create HTTP Service** form, set:

| Field | Value |
|---|---|
| **Service Contract** | `Design From Scratch` |
| **Service Base Path** | `/api/v1` |

![Create HTTP Service form with Service Contract Design From Scratch, Service Base Path /api/v1, Advanced Configurations Expand link, and Create button.](/img/genai/develop/natural-functions/27-http-service-create.png)

Click **Create**. The HTTP Service editor opens with no resources yet.

### Step 2.2 — Add the POST Resource

Click **+ Add Resource**.

![HTTP Service editor with Listener httpDefaultListener, Base Path /api/v1, the Resources section showing "No resources found. Add a new resource." and a + Add Resource button.](/img/genai/develop/natural-functions/28-http-service-add-resource.png)

In the **Select HTTP Method to Add** picker, choose **POST**.

![Select HTTP Method to Add panel with GET, POST (highlighted), PUT, DELETE, PATCH, DEFAULT.](/img/genai/develop/natural-functions/29-resource-method-picker.png)

Configure the resource:

| Field | Value |
|---|---|
| **HTTP Method** | `POST` |
| **Resource Path** | `analyze` |
| **Payload (Type / Name)** | `string` / `review` |
| **Responses** | `201` → `ReviewResponse`, `500` → `error` |

![Resource Configuration panel with HTTP Method POST, Resource Path analyze, a string review payload, and Responses 201 ReviewResponse / 500 error. Save button at the bottom right.](/img/genai/develop/natural-functions/30-resource-configuration.png)

Click **Save**. The resource flow opens with **Start**, an empty placeholder, and an **Error Handler**.

![Resource flow showing Start, an empty placeholder ("Select node from node panel."), and Error Handler.](/img/genai/develop/natural-functions/31-resource-flow-empty.png)

---

## 3. Call the Natural Function from the Resource

### Step 3.1 — Open the Add Node Panel

Click the empty placeholder between **Start** and **Error Handler**. The **Add Node** panel slides in. Expand the **AI** category and click **Call Natural Function**.

![Add Node panel with AI category expanded, showing Direct LLM (Model Provider, Call Natural Function) and RAG nodes. The Call Natural Function tile is highlighted.](/img/genai/develop/natural-functions/32-add-node-call-natural-function.png)

### Step 3.2 — Pick the Function

The **Natural Functions** picker lists every natural function in the project. Pick `analyzeCustomerReviewes`.

![Natural Functions picker with a Search box and a Current Integration section showing analyzeCustomerReviewes.](/img/genai/develop/natural-functions/33-natural-functions-picker.png)

### Step 3.3 — Bind the Argument

The configuration form opens. Each parameter on the function becomes a row; here you only have `CustomerReview`.

![Configuration form for the analyzeCustomerReviewes call: empty CustomerReview field with Text/Expression toggle, Result name reviewResponse, Variable Type ReviewResponse (locked), Save button.](/img/genai/develop/natural-functions/34-call-config-empty.png)

Bind **CustomerReview** to the inbound payload variable `review`. Leave **Result** as `reviewResponse` and **Variable Type** as the locked `ReviewResponse`.

![Configuration form filled in: CustomerReview bound to review (variable pill), Result reviewResponse, Variable Type ReviewResponse, Save button enabled.](/img/genai/develop/natural-functions/35-call-config-filled.png)

Click **Save**. The natural-function node lands in the flow.

### Step 3.4 — Add the Return

Click the empty placeholder between the natural function call and **Error Handler**. In the Add Node panel, under **Control**, pick **Return**.

![Resource flow with Start, the analyzeCustomerReviewes (reviewResponse) node, an empty placeholder, and Error Handler. The Add Node panel on the right has Return highlighted under Control.](/img/genai/develop/natural-functions/36-resource-flow-with-call.png)

In the Return panel, set the **Expression / Return value** to `reviewResponse`.

![Return node configuration panel saying "This operation has no required parameters. Optional settings can be configured below." with the Expression set to the variable reviewResponse. A Saving... indicator is on the right.](/img/genai/develop/natural-functions/37-return-node-config.png)

The completed flow:

![Final resource flow: Start → analyzeCustomerReviewes (reviewResponse) → Return reviewResponse → Error Handler.](/img/genai/develop/natural-functions/38-final-resource-flow.png)

---

## 4. Run and Test

### Step 4.1 — Run the Service

From the **Overview** page click **Run**. BI applies the `--experimental` flag and starts the service.

![Integration Overview page with the Run button highlighted in the top-right toolbar. The design diagram shows httpDefaultListener connected to the /api/v1 service with a POST /analyze resource. A toast at the bottom right reads "WSO2 default model provider configuration values were added to t…".](/img/genai/develop/natural-functions/40-design-overview-run.png)

The integrated terminal shows the build and run output:

```
Executing task: ".../bal run --experimental"
Resolving workspace dependencies
Compiling source orgs/ai_sections:0.1.0
Running executable
```

![Integrated terminal showing the bal run --experimental task output for the natural function project.](/img/genai/develop/natural-functions/41-run-terminal.png)

### Step 4.2 — Send a Test Request

From the resource editor, click **Try It** in the top-right toolbar. Send a `POST /api/v1/analyze` with this body:

```json
{
  "review": "Loved the sound quality, but the charging case died after a week and support never replied."
}
```

### Step 4.3 — Read the Structured Response

The natural function returns a fully structured response matching `ReviewResponse`:

```json
{
  "sentiment": "mixed",
  "summary": "Customer loves the sound quality but is frustrated by a faulty charging case and slow support response.",
  "topics": [
    { "name": "sound quality",     "sentiment": "positive" },
    { "name": "charging case",     "sentiment": "negative" },
    { "name": "customer support",  "sentiment": "negative" }
  ],
  "churn_risk": true,
  "suggested_action": "Reach out with a replacement case and apologize for the support delay."
}
```

The LLM analyzed the review, identified the overall sentiment, extracted per-topic sentiment, flagged the customer as a churn risk, and suggested a concrete follow-up action — all returned as a typed `ReviewResponse` record. There is no JSON parsing in your code, no schema enforcement to write by hand, and no risk of a missing field reaching the rest of the flow.

> Ballerina HTTP services return `201 Created` for POST requests by default — that's expected behaviour, not an error.

---

## What's Next

- **[Typed Return Inference](/docs/genai/develop/natural-functions/typed-return-inference)** — squeeze more accuracy out of the return type (closed records, enum unions, field descriptions).
- **[The `natural { }` Block](/docs/genai/develop/natural-functions/the-natural-block)** — write better prompt bodies.
- **[Calling from a Flow](/docs/genai/develop/natural-functions/calling-from-a-flow)** — use the function from automations, other functions, or as an agent tool.
- **[Email Generator with Direct LLM](email-generator-direct-llm.md)** — a similar tutorial built around direct LLM calls.
