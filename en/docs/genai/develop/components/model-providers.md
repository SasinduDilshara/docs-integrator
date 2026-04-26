---
sidebar_position: 2
title: Model Providers
description: Reference for every Model Provider in WSO2 Integrator BI — init form fields, advanced configurations, defaults, and supported models for the Default WSO2 provider, OpenAI, Azure OpenAI, Anthropic, Google Vertex, Mistral, DeepSeek, Ollama, and OpenRouter.
---

# Model Providers

A **Model Provider** is the connection to an LLM. Direct LLM Calls, Natural Functions, the RAG `generate` node, and AI Agents all sit on top of one. Pick a provider, fill in the form, click Save — that's it.

Every model provider exposes the same two actions, so picking a different LLM is just a connection-level swap. The rest of your flow doesn't change.

## Public Actions

Every model provider connection exposes these two actions in the right-side **Model Providers** panel.

| Action | What it does | Required parameters | Optional parameters |
|---|---|---|---|
| **Generate** | Sends a prompt to the model and binds the response to a typed Ballerina value. The everyday action behind a `generate` node, a Natural Function, or RAG `ai:generate`. | **Prompt** (the instruction template), **Expected Type** (the type the response is parsed into) | None per call — anything you'd tune (temperature, max tokens) lives on the connection. |
| **Chat** | Sends a list of chat messages and (optionally) tool definitions; returns the model's reply, including any tool calls. Used by Agents. | **Messages** (the conversation), **Tools** (tool definitions for tool calling) | **Stop** (a stop sequence). |

> Per-call overrides are not exposed in the form. Anything that varies per request belongs in the prompt; anything that varies across the project is set once on the connection (see [Advanced Configurations](#advanced-configurations) below).

## Where To Find Model Providers

Two equivalent places:

- **Add Node panel** → **AI** → **Direct LLM** → **Model Provider** (drops a Model Provider connection into a flow).
- **Right-side Model Providers panel** → **+ Add Model Provider** (adds a connection from anywhere in the project).

![Right-side Model Providers panel showing the search bar and a + Add Model Provider button at the top of an empty list.](/img/genai/develop/components/model-providers/00-panel-empty.png)

Click **+ Add Model Provider** and the **Select Model Provider** picker opens with a card for each provider type:

![Select Model Provider picker listing Default Model Provider (WSO2), Anthropic, Azure OpenAI, Deepseek, Google Vertex, Mistral, Ollama, OpenAI, with one-line descriptions for each.](/img/genai/develop/components/model-providers/01-select-list-top.png)

Scroll for the rest:

![Select Model Provider picker scrolled to show Deepseek (highlighted), Google Vertex, Mistral, Ollama, OpenAI, and OpenRouter Model Provider entries.](/img/genai/develop/components/model-providers/02-select-list-bottom.png)

## Implementations at a Glance

| Provider | Module | API key required? | Has embedding provider? |
|---|---|---|---|
| **Default WSO2** | `ballerina/ai` | No (signed-in via WSO2) | Yes — see [Embedding Providers → Default WSO2](/docs/genai/develop/components/embedding-providers#default-wso2-embedding-provider) |
| **OpenAI** | [`ballerinax/ai.openai`](https://central.ballerina.io/ballerinax/ai.openai/latest) | Yes | Yes |
| **Azure OpenAI** | [`ballerinax/ai.azure`](https://central.ballerina.io/ballerinax/ai.azure/latest) | Yes | Yes |
| **Anthropic** | [`ballerinax/ai.anthropic`](https://central.ballerina.io/ballerinax/ai.anthropic/latest) | Yes | No |
| **Google Vertex** | [`ballerinax/ai.googleapis.vertex`](https://central.ballerina.io/ballerinax/ai.googleapis.vertex/latest) | OAuth2 / service account | Yes |
| **Mistral** | [`ballerinax/ai.mistral`](https://central.ballerina.io/ballerinax/ai.mistral/latest) | Yes | No |
| **DeepSeek** | [`ballerinax/ai.deepseek`](https://central.ballerina.io/ballerinax/ai.deepseek/latest) | Yes | No |
| **Ollama** | [`ballerinax/ai.ollama`](https://central.ballerina.io/ballerinax/ai.ollama/latest) | No (local) | No |
| **OpenRouter** | [`ballerinax/ai.openrouter`](https://central.ballerina.io/ballerinax/ai.openrouter/latest) | Yes | Yes |

## Standard HTTP Advanced Configurations

Every provider with a hosted endpoint (everyone except the Default WSO2 provider, which is preconfigured) shares the **same** HTTP-level Advanced Configurations. They tune the underlying HTTP client and apply to every request the provider makes.

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | provider-specific | URL string | Override the provider's API base URL — useful for OpenAI-compatible gateways or self-hosted endpoints. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on the response length. |
| **Temperature** | `0.7` | `0.0`–`2.0` (provider-dependent) | Sampling temperature. Lower = more deterministic, higher = more creative. Use `0.0` for the most reproducible results. |
| **HTTP Version** | `HTTP_2_0` | `HTTP_1_1`, `HTTP_2_0` | The HTTP version used for requests. |
| **HTTP1 Settings** | `{}` | Record (keep-alive, version, chunking) | HTTP/1.x protocol settings. |
| **HTTP2 Settings** | `{}` | Record | HTTP/2 protocol settings. |
| **Timeout** | `60` (seconds) | Any positive number | Per-request timeout. |
| **Forwarded** | `"disable"` | `"disable"`, `"enable"`, `"transient"` | Whether to set `Forwarded` / `X-Forwarded-For` headers when behind a proxy. |
| **Pool Configuration** | `{}` | Record (max active, idle time) | Request pool settings. |
| **Cache Configuration** | `{}` | Record | HTTP response caching. |
| **Compression** | `AUTO` | `AUTO`, `ALWAYS`, `NEVER` | `accept-encoding` handling. |
| **Circuit Breaker Configuration** | `{}` | Record (`rollingWindow`, `failureThreshold`, `resetTime`, `statusCodes`) | Open the breaker on repeated failures so calls fail fast and recover. |
| **Retry Configuration** | `{}` | Record (`count`, `interval`, `backOffFactor`, `maxWaitInterval`, `statusCodes`) | Retry on failure. |
| **Response Limits** | `{}` | Record (max body, max headers) | Maximum inbound response size. |
| **Secure Socket** | `()` | Record (trust store, key store, client auth) | TLS / SSL options. |
| **Proxy** | `()` | Record (host, port, userName, password) | HTTP proxy settings. |
| **Validation** | `true` | `true`, `false` | Inbound payload validation. |

> The same set of fields appears in the **Advanced Configurations** section of every provider's Create form. Below, each provider's section only calls out provider-specific knobs that go on top.

---

## Default WSO2 Model Provider

Provided by the core `ballerina/ai` package. Routes through the WSO2 intelligence service. **No API key in your source** — a one-time WSO2 sign-in writes the credentials into your project's `Config.toml`. The fastest way to get an LLM running while you're prototyping.

### Init form

![Create Model Provider form for the Default WSO2 provider. Header reads 'Creates a default model provider based on the provided wso2ProviderConfig'. Banner: 'This is a simple operation that requires no parameters. Specify where to store the result to finish.' Two fields: Model Provider Name (default aiWso2modelprovider) and Result Type (locked to ai:Wso2ModelProvider). Save button.](/img/genai/develop/components/model-providers/wso2-default.png)

No provider-specific fields — sign in with your WSO2 account and BI handles the rest. There are no Advanced Configurations on this provider.

> When you click **Save**, the Command Palette opens with **Ballerina: Configure default WSO2 model provider**. Sign in once and the credentials are written to `Config.toml` automatically. You can re-run the command at any time to refresh.

---

## OpenAI

Talks to OpenAI's hosted models (GPT-4o, GPT-4.1, o1 reasoning models, GPT-3.5-turbo). Official: [platform.openai.com](https://platform.openai.com/).

### Init form

![Create Model Provider form for OpenAI showing two required fields: API Key (text/expression toggle) and Model Type (select/expression toggle, value 'No Selection'), then Advanced Configurations Expand link, Model Provider Name set to openaiModelprovider, Result Type set to openai:ModelProvider.](/img/genai/develop/components/model-providers/openai-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Your OpenAI API key (starts with `sk-…`). Reference a `configurable` in production. |
| **Model Type** | Yes | — | `gpt-4o`, `gpt-4o-mini`, `gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`, `gpt-4-turbo`, `gpt-3.5-turbo`, `o1`, `o1-pro`, `chatgpt-4o-latest`, `gpt-4o-audio-preview`. Date-pinned variants (e.g. `gpt-4o-2024-11-20`) are also available for reproducibility. |

### Advanced configurations

![OpenAI Create Model Provider form with Advanced Configurations expanded. Visible fields: Service URL (default https://api.openai.com/v1), Maximum Tokens (default 512), Temperature (default 0.7), HTTP Version (default http:HTTP_2_0), HTTP1 Settings, HTTP2 Settings, Timeout.](/img/genai/develop/components/model-providers/openai-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://api.openai.com/v1` | URL string | OpenAI API base URL. Override only for OpenAI-compatible gateways. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `0.7` | `0.0`–`2.0` | Sampling temperature. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations) (Timeout, Retry, Circuit Breaker, Proxy, etc.).

> The OpenAI package also ships an **Embedding Provider** — see [Embedding Providers → OpenAI](/docs/genai/develop/components/embedding-providers#openai).

---

## Azure OpenAI

Same family of models as OpenAI, hosted on Azure with per-resource deployments. Official: [Azure OpenAI Service](https://azure.microsoft.com/services/cognitive-services/openai-service/).

### Init form

![Create Model Provider form for Azure OpenAI showing four required fields: Service URL, API Key, Deployment ID, API Version. Below: Advanced Configurations Expand link, Model Provider Name azureOpenaimodelprovider, Result Type azure:OpenAiModelProvider.](/img/genai/develop/components/model-providers/azure-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Service URL** | Yes | — | Base URL of your Azure OpenAI resource, e.g. `https://your-resource.openai.azure.com`. |
| **API Key** | Yes | — | Azure OpenAI API key. |
| **Deployment ID** | Yes | — | The deployment identifier you created in the Azure portal (the model name is implicit in the deployment). |
| **API Version** | Yes | — | Azure OpenAI API version, e.g. `2023-07-01-preview`. |

### Advanced configurations

![Azure OpenAI Create Model Provider form with Advanced Configurations expanded. Visible fields: Maximum Tokens (default 512), Temperature (default 0.7), HTTP Version.](/img/genai/develop/components/model-providers/azure-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `0.7` | `0.0`–`2.0` | Sampling temperature. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

> The Azure package also ships an **Embedding Provider** and the **Azure AI Search Knowledge Base** — see [Embedding Providers → Azure OpenAI](/docs/genai/develop/components/embedding-providers#azure-openai) and [Knowledge Bases → Azure AI Search](/docs/genai/develop/components/knowledge-bases#azure-ai-search-knowledge-base).

---

## Anthropic

Anthropic's Claude family (Sonnet, Opus, Haiku). Official: [anthropic.com](https://www.anthropic.com/).

### Init form

![Create Model Provider form for Anthropic showing two required fields: API Key and Model Type. Below: Advanced Configurations Expand link, Model Provider Name anthropicModelprovider, Result Type anthropic:ModelProvider.](/img/genai/develop/components/model-providers/anthropic-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Anthropic API key. |
| **Model Type** | Yes | — | `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-opus-4-5`, `claude-sonnet-4-5`, `claude-haiku-4-5`, `claude-3-7-sonnet-20250219`, `claude-3-5-sonnet-20241022`, `claude-3-5-haiku-20241022`, `claude-3-opus-20240229`, `claude-3-sonnet-20240229`, `claude-3-haiku-20240307`. |

### Advanced configurations

![Anthropic Create Model Provider form with Advanced Configurations expanded showing Service URL (default https://api.anthropic.com/v1), Maximum Tokens (default 512), Temperature (default 0.7), HTTP Version.](/img/genai/develop/components/model-providers/anthropic-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://api.anthropic.com/v1` | URL string | Anthropic API base URL. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. **Anthropic always sends this on every call**, so the connection always carries a default. |
| **Temperature** | `0.7` | `0.0`–`1.0` | Sampling temperature. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

---

## Google Vertex AI

Vertex AI exposes Google's Gemini models alongside hosted Anthropic, Mistral, Meta, DeepSeek, and other open-weight models. Official: [Vertex AI](https://cloud.google.com/vertex-ai).

### Init form

![Create Model Provider form for Google Vertex showing three required fields: Auth (record/expression toggle, with hint 'OAuth2RefreshConfig for OAuth2 refresh token flow, or ServiceAccountConfig for automatic token refresh via service account'), Project ID, Model (with hint 'The model in publisher/model-name format, e.g., google/gemini-2.0-flash'). Below: Advanced Configurations Expand link, Model Provider Name vertexModelprovider, Result Type vertex:ModelProvider.](/img/genai/develop/components/model-providers/vertex-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Auth** | Yes | — | OAuth2 refresh-token record, a service-account record, or a path to a service-account JSON file. See [auth options](#vertex-auth-options) below. |
| **Project ID** | Yes | — | Your Google Cloud project ID. |
| **Model** | Yes | — | Publisher-prefixed model name. Examples: `google/gemini-2.0-flash`, `anthropic/claude-sonnet-4-6`, `mistralai/mistral-medium-3`, `meta/llama-4-maverick-17b-128e-instruct-maas`, `deepseek-ai/deepseek-v3-0324`, `qwen/qwen3-235b-a22b`, `kimi/kimi-k2`, `minimax/minimax-m2`. |

### Advanced configurations

![Google Vertex Create Model Provider form with Advanced Configurations expanded showing Location (default 'global'), Service URL (default 'https://\{location\}-aiplatform.googleapis.com').](/img/genai/develop/components/model-providers/vertex-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Location** | `"global"` | `"global"`, `"us-central1"`, `"europe-west1"`, etc. | Google Cloud region. |
| **Service URL** | `""` (auto-derived from Location) | URL string | Override the regional endpoint. Defaults to `https://\{location\}-aiplatform.googleapis.com`. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `()` (omitted from request) | `0.0`–`2.0` or empty | Sampling temperature. Leave empty for models that reject the field (e.g. some Anthropic-on-Vertex calls). |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

### Vertex auth options {#vertex-auth-options}

| Auth type | Fields | When to use |
|---|---|---|
| **OAuth2 refresh config** | `clientId`, `clientSecret`, `refreshToken`, `refreshUrl` | Already have OAuth2 refresh-token credentials. |
| **Service account record** | `clientEmail`, `privateKey`, optional `scopes` | Want explicit credentials in source. |
| **Service account JSON path** | A file path string | Easiest — point at the downloaded service-account JSON file from the Google Cloud console. The connector reads `client_email` and `private_key` automatically and refreshes the token. |

> Vertex also ships an **Embedding Provider** — see [Embedding Providers → Google Vertex](/docs/genai/develop/components/embedding-providers#google-vertex).

---

## Mistral

EU-hosted Mistral and Mixtral models, including Codestral and Pixtral. Official: [mistral.ai](https://www.mistral.ai/).

### Init form

![Create Model Provider form for Mistral showing two required fields: API Key and Model Type. Below: Advanced Configurations Expand link, Model Provider Name mistralModelprovider, Result Type mistral:ModelProvider.](/img/genai/develop/components/model-providers/mistral-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | Mistral AI API key. |
| **Model Type** | Yes | — | `mistral-large-latest`, `mistral-medium-latest`, `mistral-small-latest`, `codestral-latest`, `ministral-8b-latest`, `ministral-3b-latest`, `pixtral-large-latest`, `open-mixtral-8x22b`, `open-mixtral-8x7b`, `open-mistral-nemo`, `devstral-small-latest`, `mistral-saba-latest`. Date-pinned variants (e.g. `mistral-large-2411`, `codestral-2501`) are also available. |

### Advanced configurations

![Mistral Create Model Provider form with Advanced Configurations expanded showing Service URL (default https://api.mistral.ai/v1), Maximum Tokens (default 512), Temperature (default 0.7).](/img/genai/develop/components/model-providers/mistral-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://api.mistral.ai/v1` | URL string | Mistral API base URL. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `0.7` | `0.0`–`1.0` | Sampling temperature. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

---

## DeepSeek

DeepSeek's chat and reasoning models. Official: [deepseek.com](https://www.deepseek.com/).

### Init form

![Create Model Provider form for Deepseek showing one required field: API Key. Below: Advanced Configurations Expand link, Model Provider Name deepseekModelprovider, Result Type deepseek:ModelProvider.](/img/genai/develop/components/model-providers/deepseek-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | DeepSeek API key. |

### Advanced configurations

![DeepSeek Create Model Provider form with Advanced Configurations expanded showing Model Type (default DEEPSEEK_CHAT), Service URL (default https://api.deepseek.com), Maximum Token (default 512), Temperature (default 0.7), HTTP Version, HTTP1 Settings.](/img/genai/develop/components/model-providers/deepseek-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Model Type** | `deepseek-chat` | `deepseek-chat`, `deepseek-reasoner` | Which DeepSeek model to use. |
| **Service URL** | `https://api.deepseek.com` | URL string | DeepSeek API base URL. |
| **Maximum Token** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `0.7` | `0.0`–`1.0` | Sampling temperature. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

---

## Ollama

Local model runner — nothing leaves the machine. Useful for offline development, privacy-sensitive flows, and any model Ollama can pull (Llama, Mistral, Qwen, etc.). Official: [ollama.com](https://ollama.com/).

### Init form

![Create Model Provider form for Ollama showing one required field: Model Type. Below: Advanced Configurations Expand link, Model Provider Name ollamaModelprovider, Result Type ollama:ModelProvider.](/img/genai/develop/components/model-providers/ollama-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **Model Type** | Yes | — | Anything `ollama pull` can fetch (e.g. `llama3.2`, `mistral`, `qwen2.5`). |

### Advanced configurations

Ollama exposes Mirostat sampling and other decoding controls that the hosted providers don't.

![Ollama Create Model Provider form with Advanced Configurations expanded showing Service URL (default http://localhost:11434), Mirostat Sampling (default 0, options 0=disabled, 1=Mirostat, 2=Mirostat 2.0), Mirostat Eta (default 0.1), Mirostat Tau (default 5.0), Context Window Size (default 2048).](/img/genai/develop/components/model-providers/ollama-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `http://localhost:11434` | URL string | URL of the local Ollama daemon. |
| **Mirostat Sampling** | `0` | `0` (disabled), `1` (Mirostat), `2` (Mirostat 2.0) | Whether to use Mirostat sampling for controlling perplexity. |
| **Mirostat Eta** | `0.1` | Any positive number | Mirostat learning rate. Higher = more responsive to feedback. |
| **Mirostat Tau** | `5.0` | Any positive number | Mirostat target perplexity. Lower = more focused, more coherent. |
| **Context Window Size** | `2048` | Any positive integer | Context window in tokens. |
| **Repeat Last N** | `64` | `0` (disabled), `-1` (=context window), or a positive integer | Look-back window for repetition penalty. |
| **Repeat Penalty** | `1.1` | Any positive number | Repetition penalty strength. Higher = stronger penalty. |
| **Temperature** | `0.8` | Any positive number | Sampling temperature. |
| **Seed** | `0` | Any integer | Random seed for deterministic generation. |
| **Number of Predictions** | `-1` | `-1` (unlimited) or any positive integer | Maximum tokens to generate. |
| **Top K** | `40` | Any positive integer | Top-K sampling. |
| **Top P** | `0.9` | `0.0`–`1.0` | Top-P (nucleus) sampling. |
| **Min P** | `0.0` | `0.0`–`1.0` | Minimum probability filter relative to the top token. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

> Ollama is the only provider with no API key — authentication is implicit because the daemon trusts callers on `localhost`.

---

## OpenRouter

OpenRouter routes a single API across many model providers (OpenAI, Anthropic, Mistral, Meta, Cohere, …). Use it when you want one key and the freedom to swap models by string. Official: [openrouter.ai](https://openrouter.ai/).

### Init form

![Create Model Provider form for OpenRouter showing two required fields: API Key (with link to https://openrouter.ai/keys) and Model Type (with example values 'openai/gpt-4o', 'anthropic/claude-3.5-sonnet'). Below: Advanced Configurations Expand link, Model Provider Name openrouterModelprovider, Result Type openrouter:ModelProvider.](/img/genai/develop/components/model-providers/openrouter-basic.png)

| Field | Required | Default | Available values |
|---|---|---|---|
| **API Key** | Yes | — | OpenRouter API key. Get one from [openrouter.ai/keys](https://openrouter.ai/keys). |
| **Model Type** | Yes | — | Qualified model name. Examples: `openai/gpt-4o`, `anthropic/claude-3.5-sonnet`, `mistralai/mistral-large`, `meta-llama/llama-3.1-70b-instruct`. See OpenRouter's [model list](https://openrouter.ai/models). |

### Advanced configurations

![OpenRouter Create Model Provider form with Advanced Configurations expanded showing Service URL (default https://openrouter.ai/api/v1), Site URL (sent as HTTP-Referer header), Site Name (sent as X-OpenRouter-Title header), Maximum Tokens (default 512), Temperature (default 0.7), HTTP Version.](/img/genai/develop/components/model-providers/openrouter-advanced.png)

| Field | Default | Available values | What it controls |
|---|---|---|---|
| **Service URL** | `https://openrouter.ai/api/v1` | URL string | OpenRouter API base URL. |
| **Site URL** | `()` | URL string or empty | Optional site URL sent as the `HTTP-Referer` header. Used by OpenRouter for site attribution and leaderboards. |
| **Site Name** | `()` | String or empty | Optional site name sent as the `X-OpenRouter-Title` header. |
| **Maximum Tokens** | `512` | Any positive integer | Hard cap on response length. |
| **Temperature** | `()` (omitted) | `0.0`–`2.0` or empty | Sampling temperature. Leave empty to omit. |

Plus the [Standard HTTP Advanced Configurations](#standard-http-advanced-configurations).

> The OpenRouter package also ships an **Embedding Provider** — see [Embedding Providers → OpenRouter](/docs/genai/develop/components/embedding-providers#openrouter).

---

## Where Providers Live (After Creation)

Once you click **Save**, the provider becomes a **connection** in your project and shows up in three places at once:

- The left **Connections** tree, under your project (e.g. `wso2ModelProvider`, `openaiModelprovider`).
- The **Model Providers** panel on the right side of any flow editor.
- Wherever a node asks for a model — a `generate` node, a natural function, or the **Model** field of an agent.

The right-side **Model Providers** panel lists every provider connection in the project, with a **+** button to add another and a chevron to expand each connection's available actions:

![The Model Providers right-side panel listing four model-provider connections — anthropicModelprovider, azureOpenaimodelprovider, openaiModelprovider, wso2ModelProvider — each with a chevron and provider logo.](/img/genai/develop/agents/25-model-providers-panel-multi.png)

At the project level, every provider also appears in the left **Connections** tree, and the integration project's **Design** view wires each artifact to the provider it depends on:

![The integration project Design overview with the left sidebar Connections tree populated with four model-provider connections, and the main canvas wiring three artifacts (chat agent service, HTTP service, MCP service) to their respective model-provider nodes on the right with provider logos.](/img/genai/develop/agents/26-project-design-multi-providers.png)

## Editing or Replacing a Provider

To change a provider's API key, model name, or any other field after it's been created, click the provider name in the left **Connections** tree. The **Edit Connection** modal opens:

![Edit Connection modal centered on screen with Variable Name field, Variable Type field with edit pencil icon, Update Connection button at the bottom.](/img/genai/develop/shared/10-edit-connection-modal.png)

| Field | What it does |
|---|---|
| **Variable Name** | Rename the connection. Updates every reference automatically. |
| **Variable Type** | The Ballerina type. Click the pencil icon to change it (e.g. swap one provider implementation for another — OpenAI → Azure OpenAI — without renaming the connection in the rest of your code). |
| **Advanced Configurations** | Expand to edit HTTP and model parameters. |
| **Update Connection** | Save the change. Existing nodes that referenced the connection continue to work. |

> Editing a connection follows the same pattern for every component type — embedding providers, vector stores, knowledge bases, chunkers, and memory connections all use the same Edit Connection modal.

## What's Next

- **[Embedding Providers](/docs/genai/develop/components/embedding-providers)** — vector embeddings for RAG. The OpenAI, Azure, Vertex, OpenRouter, and Default WSO2 packages also ship embedding providers.
- **[Direct LLM Calls](/docs/genai/develop/direct-llm/overview)** — how the BI canvas wires up a model provider and the `generate` node.
- **[Agents](/docs/genai/develop/agents/overview)** — model providers are one of three required pieces of an agent (alongside [Tools](/docs/genai/develop/agents/tools) and [Memory](/docs/genai/develop/components/memory)).
