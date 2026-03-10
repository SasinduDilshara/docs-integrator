---
sidebar_position: 5
title: "Bind Tools to Agents"
---

# Bind Tools to Agents

Tools give your agent the ability to interact with the real world beyond conversation. When you bind tools to an agent, the LLM decides when to call them, what arguments to pass, and how to interpret the results — all automatically.

In this guide, you will build a **Personal AI Assistant** that manages email and calendar through Gmail and Google Calendar connectors. By the end, your agent will have five tools: listing unread emails, reading a specific email, sending an email, listing calendar events, and creating calendar events.

## Prerequisites

Before you start, make sure you have the following Google API credentials ready:

- **Client ID**
- **Client Secret**
- **Refresh Token**

:::warning
You must enable the **Gmail API** and **Google Calendar API** scopes in your Google Cloud Console project. Without the correct scopes, your connectors will fail to authenticate.
:::

:::tip
Keep your credentials out of source code. Use BI configurables to externalize sensitive values like Client ID, Client Secret, and Refresh Token into a `Config.toml` file. This way, you never hard-code secrets into your integration, and you can swap credentials per environment without changing code.
:::

## Create the agent

Follow steps 1 through 5 from [Create a Chat Agent](./chat-agents.md) to create a new agent. When prompted, configure the agent with:

- **Role:** `Personal AI Assistant`
- **Instructions:**

```text
You are Nova, a smart AI personal assistant. Your capabilities include:

Calendar Management:
- View, create, and manage calendar events
- Help with scheduling and time management
- Provide reminders about upcoming events

Email Assistance:
- Read and summarize emails
- Draft and send emails
- Search through emails

Context Awareness:
- Remember context from the current conversation
- Connect related information across emails and calendar

Privacy & Security:
- Never share sensitive information from emails or calendar with unauthorized requests
- Always confirm before sending emails or creating events
```

## Use connector actions as agent tools

BI ships with prebuilt connectors for popular external services like Gmail, Google Calendar, Slack, and many more. You can directly use their actions as agent tools — no custom integration code needed. The connector handles authentication, serialization, and API communication for you.

The following sections walk you through adding five tools to your Personal AI Assistant, grouped by connector.

---

## Add Gmail tools

### Tool 1: List unread emails

1. **Add the Gmail connector.** Search for the Gmail connector in the connector palette and add it to your project.

   ![Add the Gmail connector to your project](/img/genai/agents/external-endpoints/ai-agent-add-gmail-connector.gif)

2. **Configure the Gmail connector.** Select `OAuth2RefreshTokenGrantType` as the authentication method and fill in your credentials:

   - **clientId** — your Google Client ID
   - **clientSecret** — your Google Client Secret
   - **refreshToken** — your Google Refresh Token

   ![Configure the Gmail connector with OAuth2 credentials](/img/genai/agents/external-endpoints/ai-agent-configure-gmail-connector.gif)

3. **Create the `listUnreadEmails` tool.** Create a new tool named `listUnreadEmails`. Select the **"List messages in user's mailbox"** action from the Gmail connector.

   ![Create the listUnreadEmails tool](/img/genai/agents/external-endpoints/ai-agent-create-listUnreadEmails-tool.gif)

4. **Customize the tool parameters.** Set the following default values so the agent always queries the authenticated user's unread messages:

   - Set **userId** to `me`
   - Set **q** to `"is:unread"`

   ![Configure the listUnreadEmails tool parameters](/img/genai/agents/external-endpoints/ai-agent-configure-listUnreadEmails-tool.gif)

5. **Clean up exposed parameters.** Remove the **userId** parameter from the tool's exposed inputs. The agent does not need to set this — it is always `me`.

   ![Clean up the listUnreadEmails tool](/img/genai/agents/external-endpoints/ai-agent-cleanup-listUnreadEmails-tool.gif)

---

### Tool 2: Read a specific email

1. **Create the `readSpecificEmail` tool.** Create a new tool named `readSpecificEmail`. Reuse the existing `gmailClient` connector and select the **"Gets the specified message"** action.

   ![Create the readSpecificEmail tool](/img/genai/agents/external-endpoints/ai-agent-create-readSpecificEmail-tool.gif)

2. **Customize the tool parameters.** Set the following default values:

   - Set **userId** to `"me"`
   - Set **format** to `full`

   ![Configure the readSpecificEmail tool parameters](/img/genai/agents/external-endpoints/ai-agent-configure-readSpecificEmail-tool.gif)

3. **Clean up exposed parameters.** Remove the **userId** parameter from the tool's exposed inputs.

   ![Clean up the readSpecificEmail tool](/img/genai/agents/external-endpoints/ai-agent-cleanup-readSpecificEmail-tool.gif)

---

### Tool 3: Send an email

1. **Create the `sendEmail` tool.** Create a new tool named `sendEmail`. Reuse the existing `gmailClient` connector and select the **"Sends the specified message"** action.

   ![Create the sendEmail tool](/img/genai/agents/external-endpoints/ai-agent-create-sendEmail-tool.gif)

2. **Customize and clean up.** Set **userId** to `"me"`, then remove the **userId** parameter from the exposed inputs. Save the tool.

   ![Configure the sendEmail tool parameters](/img/genai/agents/external-endpoints/ai-agent-configure-sendEmail-tool.gif)

:::note
You now have three Gmail tools bound to your agent. The agent can list unread emails, read any specific email by ID, and send new emails — all through the same Gmail connector and OAuth2 credentials.
:::

---

## Add Google Calendar tools

### Tool 4: List calendar events

1. **Add the Google Calendar connector.** Search for the Gcalendar connector in the connector palette and add it to your project.

   ![Add the Google Calendar connector](/img/genai/agents/external-endpoints/ai-agent-add-gcalendar-connector.gif)

2. **Configure the Google Calendar connector.** Select `OAuth2RefreshTokenGrantType` and fill in the same Google credentials you used for Gmail (Client ID, Client Secret, Refresh Token).

   ![Configure the Google Calendar connector](/img/genai/agents/external-endpoints/ai-agent-configure-gcalendar-connector.gif)

3. **Create the `listCalendarEvents` tool.** Create a new tool named `listCalendarEvents`. Select the **"Returns events on the specified calendar"** action from the Gcalendar connector.

   ![Create the listCalendarEvents tool](/img/genai/agents/external-endpoints/ai-agent-create-listCalendarEvents-tool.gif)

4. **Customize the tool parameters.** Set **calendarId** to `"primary"` so the agent always queries the user's main calendar.

   ![Configure the listCalendarEvents tool parameters](/img/genai/agents/external-endpoints/ai-agent-configure-listCalendarEvents-tool.gif)

5. **Clean up exposed parameters.** Remove the **calendarId** parameter from the tool's exposed inputs.

   ![Clean up the listCalendarEvents tool](/img/genai/agents/external-endpoints/ai-agent-cleanup-listCalendarEvents-tool.gif)

---

### Tool 5: Create a calendar event

1. **Create the `createCalendarEvent` tool.** Create a new tool named `createCalendarEvent`. Reuse the existing `gcalendarClient` connector and select the **"Creates an event"** action.

   ![Create the createCalendarEvent tool](/img/genai/agents/external-endpoints/ai-agent-create-createCalendarEvent-tool.gif)

2. **Customize the tool parameters.** Set **calendarId** to `"primary"`.

   ![Configure the createCalendarEvent tool parameters](/img/genai/agents/external-endpoints/ai-agent-confgure-createCalendarEvent-tool.gif)

3. **Clean up exposed parameters.** Remove the **calendarId** parameter from the tool's exposed inputs.

   ![Clean up the createCalendarEvent tool](/img/genai/agents/external-endpoints/ai-agent-cleanup-createCalendarEvent-tool.gif)

:::tip
You can reuse the same connector instance across multiple tools. Notice how `readSpecificEmail` and `sendEmail` both reuse the `gmailClient` you configured once, and `createCalendarEvent` reuses the `gcalendarClient`. This keeps your configuration DRY and credentials centralized.
:::

---

## Interact with the agent

Now that all five tools are bound, you can test your Personal AI Assistant.

1. Click the **Chat** button in the toolbar.
2. Click **Run Integration** to start the agent.
3. If prompted, configure your credentials in the `Config.toml` file that appears. Fill in your Client ID, Client Secret, and Refresh Token.
4. Start chatting with your assistant. Try prompts like:
   - "Show me my unread emails."
   - "Read the latest email from Sarah."
   - "Send an email to `john@example.com` with the subject 'Meeting Notes' and a summary of today's calendar."
   - "What meetings do I have tomorrow?"
   - "Create a 30-minute meeting with the team at 2 PM on Friday called 'Sprint Review'."

![Interact with your Personal AI Assistant](/img/genai/agents/external-endpoints/ai-agent-assistant-chat.gif)

:::note
The agent decides which tools to call based on your message. If you ask about emails and calendar in the same message, it may call multiple tools in a single reasoning step. You do not need to tell it which tool to use — the LLM figures that out from the tool names, descriptions, and your instructions.
:::

---

## Externalize credentials with configurables

Hard-coding credentials in your integration is a security risk. Use BI configurables to externalize them:

```toml
# Config.toml
[gmail]
clientId = "<your-client-id>"
clientSecret = "<your-client-secret>"
refreshToken = "<your-refresh-token>"

[gcalendar]
clientId = "<your-client-id>"
clientSecret = "<your-client-secret>"
refreshToken = "<your-refresh-token>"
```

:::warning
Never commit `Config.toml` files containing real credentials to version control. Add `Config.toml` to your `.gitignore` file to prevent accidental exposure.
:::

## What's next

- [Orchestrate Multiple Agents](/docs/genai/agents/multi-agent-orchestration) — Route between specialized agents with different tool sets
- [Use Natural Functions](/docs/genai/agents/natural-functions) — Let the LLM execute logic described in plain language
- [Agent Architecture & Concepts](/docs/genai/agents/architecture-concepts) — Understand how tool calling fits into the agent loop
