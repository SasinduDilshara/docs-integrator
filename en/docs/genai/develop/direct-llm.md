---
sidebar_position: 1
title: Direct LLM Calls
description: Step-by-step visual guide to building a direct LLM call in WSO2 Integrator -- create an HTTP service that generates professional emails using an LLM.
---

# Direct LLM Calls

A **Large Language Model (LLM)** is an AI model trained on vast amounts of text that can understand and generate human language. You can think of it as an intelligent assistant that can summarize, classify, extract information, or generate content — all from a simple text instruction called a **prompt**.

WSO2 Integrator lets you connect to an LLM **model provider** (such as OpenAI, Anthropic, or WSO2's built-in default provider), send a prompt describing what you need, and receive a structured response that your integration can use immediately — all without writing any code.

This guide walks you through building an **HTTP service that generates professional emails using an LLM**:

1. **[Configuring LLM Providers](#1-configuring-llm-providers)** — Create the HTTP service, define request and response types, and add a model provider.
2. **[Constructing Prompts](#2-constructing-prompts)** — Add a generate node and write the email generation prompt.
3. **[Handling Responses](#3-handling-responses)** — Bind the response, complete the flow, and test the service.

---

## 1. Configuring LLM Providers

In this section you will create an HTTP service with a resource for generating emails, define the request and response types, and add a model provider.

### Step 1.1: Create an HTTP Service

1. From the left sidebar, open the **Artifacts** page.
2. Under **Integration as API**, click **HTTP Service**.

![Artifacts page with HTTP Service highlighted under Integration as API](/img/genai/develop/direct-llm/01-artifacts-page.png)

3. In the **Create HTTP Service** form, select **Design From Scratch**, enter `/api/v1` as the **Service Base Path**, and click the blue **Create** button.

![Create HTTP Service form with Service Base Path set to /api/v1](/img/genai/develop/direct-llm/02-create-http-service.png)

### Step 1.2: Define the Request Payload

After the service is created, the **New Resource Configuration** panel opens. Set the **HTTP Method** to `POST` and the **Resource Path** to `/emails/generate`.

Next, define the request payload type. Click **Define Payload** and in the dialog that opens:

1. Select the **Import** tab.
2. Paste the following sample JSON into the **Sample data** field:

   ```json
   {
     "recipientName": "Sarah Fernando",
     "senderName": "John Perera",
     "timeSlots": ["2026-01-15 10:00 AM", "2026-01-16 02:00 PM"],
     "intent": "Discuss a new website project"
   }
   ```

3. Set the **Type Name** to `EmailGeneratePayload`.
4. Click **Import Type**.

![Define Payload dialog with sample JSON and type name EmailGeneratePayload](/img/genai/develop/direct-llm/03-define-payload.png)

This will create a type with the following structure:

| Field | Type | Description |
|---|---|---|
| `recipientName` | `string` | Name of the email recipient. |
| `senderName` | `string` | Name of the email sender. |
| `timeSlots` | `string[]` | Proposed meeting time slots. |
| `intent` | `string` | Purpose of the email. |

### Step 1.3: Define the Response Type

Still in the resource configuration, define the response type for the `200` status code. Click the response type selector and choose **Create New Type**:

1. In the **Create New Type** dialog, switch to the **Import** tab.
2. Set the **Name** to `EmailGenerateResponse`.
3. Paste the following JSON:

   ```json
   {
     "subject": "Meeting Request - New Website Project",
     "content": "Dear Sarah, I hope this message finds you well..."
   }
   ```

4. Click **Import**.

![Create New Type dialog with EmailGenerateResponse and sample JSON](/img/genai/develop/direct-llm/04-define-response-type.png)

This will create a type with the following structure:

| Field | Type | Description |
|---|---|---|
| `subject` | `string` | The generated email subject line. |
| `content` | `string` | The generated email body text. |

Once both types are defined, the resource configuration should show `EmailGeneratePayload` as the request payload and `EmailGenerateResponse` as the `200` response type. Click **Save**.

![Completed resource configuration with both types assigned](/img/genai/develop/direct-llm/05-resource-complete.png)

### Step 1.4: Add a Model Provider

After saving the resource, the flow editor opens. Click the **+** button between **Start** and **Error Handler**. In the **Add Node** panel, scroll to the **AI** section under **Direct LLM** and click **Model Provider**. The side panel lists all supported providers.

![Model Provider list showing Default WSO2, Anthropic, Azure OpenAI, DeepSeek, Google Vertex, Mistral, Ollama, OpenAI, and OpenRouter](/img/genai/develop/direct-llm/06-model-provider-list.png)

Click **Default Model Provider (WSO2)**. In the configuration form:

1. Set the **Model Provider Name** to `emailGenerator`.
2. Leave the **Result Type** as the default.
3. Click **Save**.

![Model Provider configuration with name emailGenerator](/img/genai/develop/direct-llm/07-model-provider-config.png)

---

## 2. Constructing Prompts

With the model provider in place, the next step is to add a **generate** node and write the prompt that instructs the LLM to compose a professional email.

### Step 2.1: Add a `generate` Node

In the **Model Providers** panel on the right, expand the **emailGenerator** section. Two actions are available: **Chat** (multi-turn conversations) and **Generate** (single one-shot prompt). Click **Generate**.

![emailGenerator expanded with Chat and Generate actions](/img/genai/develop/direct-llm/08-generate-action.png)

### Step 2.2: Write the Prompt

The **emailGenerator > generate** configuration form opens. Click the **Prompt** field to open the rich text prompt editor and enter the following instruction:

> You are an email writing assistant. Write a short email to a potential client asking for a 30-minute meeting next week to discuss a new project. Suggest two possible time slots and ask them to pick one. Keep it under 150 words and use a polite, professional tone.

![Prompt editor with the email writing assistant instruction](/img/genai/develop/direct-llm/09-prompt-editor.png)

A well-structured prompt gives the model clear direction:

| | |
|---|---|
| **Role** | *"You are an email writing assistant."* |
| **Task** | *"Write a short email to a potential client asking for a 30-minute meeting next week to discuss a new project. Suggest two possible time slots and ask them to pick one. Keep it under 150 words and use a polite, professional tone."* |

Do not click **Save** yet — the **Result** and **Expected Type** fields still need to be filled in. Continue to Section 3.

---

## 3. Handling Responses

In this section you will complete the generate form, add a return step, and test the service end to end.

### Step 3.1: Bind the Result and Save

In the **emailGenerator > generate** form, fill in the remaining two fields:

| Field | Value | Description |
|---|---|---|
| **Result** | `generatedEmail` | The variable that holds the model's response. |
| **Expected Type** | `EmailGenerateResponse` | The record type you created in Step 1.3. |

Click the blue **Save** button.

![emailGenerator > generate form with prompt, Result generatedEmail, and Expected Type EmailGenerateResponse](/img/genai/develop/direct-llm/10-generate-config.png)

For simplicity, you could use `string` as the **Expected Type** for plain text responses. However, in this walkthrough you use the `EmailGenerateResponse` record type, which contains `subject` and `content` fields. WSO2 Integrator will automatically convert the direct LLM output into the expected type, giving you a structured, typed response.

### Step 3.2: Add a Return Step

The flow now has the `ai:generate` node connected to the `emailGenerator` provider. To return the generated email as the HTTP response:

1. Click the **+** button below the `ai:generate` node.
2. In the **Add Node** panel, under **Control**, select **Return**.

![Add Node panel showing Connections, Statement, Control, and AI categories](/img/genai/develop/direct-llm/11-add-node-panel.png)

3. In the **Return** configuration, set the expression to `generatedEmail` and click **Save**.

The completed flow should show three nodes between **Start** and **Error Handler**: the `ai:generate` node (labelled `generatedEmail`) connected to the `emailGenerator` provider, and a **Return** node below it.

![Completed flow with ai:generate connected to emailGenerator and a Return node below](/img/genai/develop/direct-llm/12-flow-complete.png)

### Step 3.3: Run and Test the Service

1. Click the **Try It** button in the top-right corner of the editor. When prompted, click **Run Integration** to start the Ballerina service.
2. The **Try Service** panel opens, showing the `POST /emails/generate` endpoint with the expected request schema.

![Try Service panel showing POST /emails/generate with request body schema](/img/genai/develop/direct-llm/13-try-service.png)

3. Fill in the request body with sample values:

   ```json
   {
     "recipientName": "Jane Doe",
     "senderName": "James Smith",
     "timeSlots": ["2026-01-18 10:00 AM", "2026-01-21 11:00 AM"],
     "intent": "Discuss a new project"
   }
   ```

4. Click **Run** to send the request.

### Step 3.4: Read the Output

The service responds with a `201 Created` status and a JSON body containing the LLM-generated email:

```json
{
  "subject": "Request for Meeting to Discuss New Project",
  "content": "Dear Jane Doe,\nI hope this message finds you well. I am reaching out to see if we could schedule a 30-minute meeting next week to discuss an exciting new project..."
}
```

![Response showing 200 Created with the generated email subject and content](/img/genai/develop/direct-llm/14-response.png)

The LLM produced a complete, professionally written email — with a subject line and body — structured exactly as the `EmailGenerateResponse` type defined in Step 1.3.

---

## What's Next

- **[Defining Natural Functions](/docs/genai/develop/natural-functions/defining)** -- Wrap LLM calls in reusable functions using the natural expression syntax.
- **[Creating an Agent](/docs/genai/develop/agents/creating-agent)** -- Add tools and memory on top of direct LLM calls to build an autonomous agent.
- **[What is an LLM?](/docs/genai/key-concepts/what-is-llm)** -- A conceptual overview of how Large Language Models work.
