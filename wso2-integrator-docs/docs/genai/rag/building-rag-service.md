---
sidebar_position: 5
title: "Build a RAG Service"
description: "Build a retrieval-augmented generation service using built-in RAG components or custom external services like Pinecone and Azure OpenAI."
---

# Build a RAG Service

Once you have ingested documents into a vector store, you need a service that accepts user questions, retrieves relevant context, and returns LLM-generated answers grounded in your data. This guide covers two approaches to building that service: the built-in RAG components for quick prototyping, and a custom pipeline with external services for production deployments.

## What you will build

An HTTP service that:

1. Accepts a question from the client
2. Retrieves relevant context from a vector knowledge base
3. Augments the question with the retrieved context
4. Sends the augmented prompt to an LLM
5. Returns the generated response to the client

---

## Approach 1: Using built-in RAG components

This approach uses WSO2 Integrator's built-in knowledge base, augmentation, and model provider components. It is the fastest way to stand up a RAG query endpoint and works with the same in-memory vector store you set up during ingestion.

:::warning Prerequisites
You must complete the [RAG Ingestion Tutorial](ingestion-pipelines.md) before starting this section. The in-memory vector store is session-scoped, so the query service **must run inside the same project** you used for ingestion.
:::

:::info Using the same project
Since this approach uses an **in-memory vector store** (from the ingestion tutorial), the ingested data is only available within the same project and runtime session. Add the HTTP service to the same `rag_ingestion` project.

If you used an external vector store like Pinecone or Weaviate, you can create a separate project and configure the same external vector store connection.
:::

### Step 1: Open the existing project

Open the `rag_ingestion` project you created during the ingestion tutorial. Because the in-memory vector store only lives for the duration of the running application, both the ingestion logic and the query service need to coexist in a single project.

### Step 2: Create an HTTP service

Add a new HTTP service to the project that will expose the query endpoint.

1. In the design view, click on the **Add Artifact** button.
2. Select **HTTP Service** under the **Integration as API** category.
3. Choose **Create and use the default HTTP listener** from the **Listener** dropdown.
4. Select **Design from Scratch** as the **Service contract** option.
5. Specify the **Service base path** as `/`.
6. Click **Create** to create the service.

![Create an HTTP service](/img/genai/rag/query/create-an-http-service.gif)

### Step 3: Update the resource method

Configure the resource method to accept user queries.

1. The service will have a default resource named `greeting` with the **GET** method. Click the edit button next to the `/greeting` resource.
2. Change the HTTP method to **POST**.
3. Rename the resource to `query`.
4. Add a payload parameter named `userQuery` of type `string`.
5. Keep others set to defaults.
6. Click **Save** to apply the changes.

![Update the resource method](/img/genai/rag/query/update-the-resource-method.gif)

### Step 4: Retrieve data from the knowledge base

Since you are working in the same project where you completed the ingestion tutorial, the vector knowledge base `knowledgeBase` is already available.

1. Click on the newly created `POST` resource to open it in the flow diagram view.
2. Hover over the flow line and click the **+** icon.
3. Select **Knowledge Bases** under the **AI** section.
4. In the **Knowledge Bases** section, click on `knowledgeBase`.
5. Click on **retrieve** to open the configuration panel.
6. Set the **Query** input to the `userQuery` variable.
7. Set the **Result** to `context` to store the matched chunks.
8. Click **Save** to complete the retrieval step.

![Retrieve data from the knowledge base](/img/genai/rag/query/retrieve-data-from-knowledge-base.gif)

:::note
The retrieve action embeds the user query, searches the vector store for the most relevant chunks, and returns them as structured context -- all in a single step.
:::

### Step 5: Augment the user query with retrieved content

WSO2 Integrator includes a built-in function to augment the user query with retrieved context. This combines the original question with the relevant document chunks so the LLM has grounding data.

1. Hover over the flow line and click the **+** icon.
2. Select **Augment Query** under the **AI** section.
3. Set **Context** to `context`.
4. Set **Query** to `userQuery`.
5. Set **Result** to `augmentedUserMsg`.
6. Click **Save** to complete the augmentation step.

![Augment the user query](/img/genai/rag/query/augment-user-query.gif)

### Step 6: Connect to an LLM provider

Add a model provider that the service will use to generate answers.

1. Hover over the flow line and click the **+** icon.
2. Select **Model Provider** under the **AI** section.
3. Click **+ Add Model Provider** to create a new instance.
4. Select **Default Model Provider (WSO2)** for this tutorial.
5. Set the **Model Provider Name** to `defaultModel`.
6. Click **Save** to complete the configuration.

![Connect to an LLM provider](/img/genai/rag/query/create-a-model-provider.gif)

:::tip Model flexibility
The default WSO2 model provider gives you instant access without managing API keys. When you are ready for production, you can swap it for **OpenAI**, **Anthropic**, **Azure OpenAI**, **Google PaLM**, or even a **local model** -- the rest of the service logic stays exactly the same.
:::

### Step 7: Generate the response

Call the LLM with the augmented prompt to produce an answer.

1. Click on the `defaultModel` under the **Model Providers** section in the side panel.
2. Select the **generate** action.
3. Set the **Prompt** to the expression: `check augmentedUserMsg.content.ensureType()`.
4. Set the **Result** variable to `response`.
5. Set the **Expected Type** to `string`.
6. Click **Save**.

![Generate the response](/img/genai/rag/query/call-model-provider-generate-action.gif)

:::note Understanding the expression
The expression `check augmentedUserMsg.content.ensureType()` safely extracts the string content from the augmented message. The `check` keyword propagates any type-conversion error, while `ensureType()` performs a runtime type assertion to ensure the content is in the correct string format the LLM expects.
:::

### Step 8: Return the response

Send the LLM-generated answer back to the HTTP client.

1. Hover over the flow line and click the **+** icon.
2. Under the **Control** section, click on **Return**.
3. Set **Expression** to `response`.

![Return the response](/img/genai/rag/query/return-the-response-from-service-resource.gif)

### Step 9: Configure and run

1. Press `Ctrl/Cmd + Shift + P` and run the command: **Ballerina: Configure default WSO2 model provider**.
2. Click the **Run** button in the top right corner to start the integration.
3. The ingestion automation will run first, followed by the HTTP service.
4. Test the service:

```bash
curl -X POST http://localhost:9090/query \
  -H "Content-Type: application/json" \
  -d '"Who should I contact for refund approval?"'
```

![Configure and run the integration](/img/genai/rag/query/run-the-integration.gif)

:::warning Response may vary
Since this integration involves an LLM call, response values may not always be identical across different executions. The LLM generates answers based on the retrieved context, but phrasing and detail level can differ each time.
:::

---

## Approach 2: Custom RAG with external services

This approach builds a production-grade RAG pipeline using **Pinecone** as the vector database and **Azure OpenAI** for both embeddings and text generation. Use this when you need persistent vector storage, fine-grained control over each pipeline stage, or integration with enterprise cloud services.

:::warning Prerequisites
Before you start, make sure you have the following:

- **Pinecone** account with an API key and index URL
- **Azure OpenAI** deployment with an API key and endpoint (both an embeddings model and a chat model)
- **Devant** (or another ingestion mechanism) to populate your Pinecone index with documents
:::

:::tip Model flexibility
While this section demonstrates Azure OpenAI integration, the same principles apply to other providers -- OpenAI API, Anthropic's Claude, Google's PaLM, or local models via Ollama. Simply replace the connector and adjust the API configuration.
:::

### Step 1: Create an HTTP service

Set up a new integration project and HTTP service.

1. Create a new project or open an existing one.
2. Add an HTTP service on port **9090** with base path `/personalAssistant`.
3. Create a **POST** resource named `chat`.
4. Add a query parameter `request` of type `ChatRequestMessage`.
5. Set the return type to `string`.

![Create a new service](/img/genai/rag/build-rag/1.create-new-service.gif)

![Create a new resource](/img/genai/rag/build-rag/2.create-new-resource.gif)

### Step 2: Implement the pipeline functions

The custom RAG pipeline consists of four discrete functions, each responsible for one stage of the process.

#### getEmbeddings -- Convert text to vectors

Create a function that calls the Azure OpenAI embeddings API to convert a text string into a vector representation.

1. Set up an **Azure OpenAI embeddings connector** with your API key and endpoint.
2. The function accepts a string and returns a `float[]` vector.

![Create embeddings](/img/genai/rag/build-rag/3.create-embeddings.gif)

![Create embeddings connection](/img/genai/rag/build-rag/4.create-embeddings-connection.gif)

#### retrieveData -- Search the vector store

Create a function that sends the embedding vector to Pinecone and retrieves the most relevant document chunks.

1. Set up a **Pinecone vector connector** with your API key and index URL.
2. Query the index with the embedding vector and return the matching documents with their metadata.

![Create Pinecone connection](/img/genai/rag/build-rag/5.create-pinecone-connection.gif)

![Create retriever function](/img/genai/rag/build-rag/6.create-retriever-function.gif)

![Retriever function logic](/img/genai/rag/build-rag/7.retriever-function-logic.gif)

#### augment -- Build the grounded prompt

Create a function that takes the retrieved matches and the original user question, then assembles an augmented prompt containing the relevant context.

![Augment function logic](/img/genai/rag/build-rag/8.augment-function-logic.gif)

:::tip
Keep your augmentation prompt specific and instructive. Tell the LLM to stay within the provided context and to admit when it does not know -- this reduces hallucination and builds user trust.
:::

#### generateText -- Call the LLM

Create a function that sends the augmented prompt to Azure OpenAI's chat completion endpoint and returns the generated text.

1. Set up an **Azure OpenAI chat connector** with your API key and deployment details.

![Create chat client](/img/genai/rag/build-rag/9.create-chat-client.gif)

### Step 3: Orchestrate with llmChat

Create a single `llmChat` function that chains all four stages together:

1. **Embed** the user message using `getEmbeddings`.
2. **Retrieve** relevant documents from Pinecone using `retrieveData`.
3. **Augment** the prompt with retrieved context using `augment`.
4. **Generate** the final answer using `generateText`.

![Create LLM chat](/img/genai/rag/build-rag/10.create-llm-chat.gif)

![LLM chat logic](/img/genai/rag/build-rag/11.llm-chat-logic.gif)

### Step 4: Integrate with the HTTP service

Wire the `llmChat` function into the `chat` resource of your HTTP service so that incoming requests flow through the full RAG pipeline.

![Chat service logic](/img/genai/rag/build-rag/12.chat-service-logic.gif)

### Step 5: Run and test

Run the integration and send a test query:

![Execute the RAG service](/img/genai/rag/build-rag/13.rag-execute.gif)

```bash
curl --location 'http://localhost:9090/personalAssistant/chat' \
  --header 'Content-Type: application/json' \
  --data '{"message": "What is the process for reporting safety concerns?"}'
```

The response is a natural-language answer generated by Azure OpenAI, grounded in the documents stored in your Pinecone index.

---

## Comparing the two approaches

| Aspect | Built-in components | Custom external services |
|---|---|---|
| **Setup time** | Minutes | Hours |
| **Vector store** | In-memory (session-scoped) | Pinecone (persistent) |
| **Embedding provider** | Built-in | Azure OpenAI |
| **LLM provider** | WSO2 default (swappable) | Azure OpenAI |
| **Best for** | Prototyping, tutorials, demos | Production, enterprise deployments |
| **Data persistence** | Lost on restart | Persisted in Pinecone |

:::tip
Start with Approach 1 to validate your RAG logic quickly, then migrate to Approach 2 when you need persistent storage, custom embedding models, or enterprise-grade infrastructure.
:::

## Adding a system prompt for domain context

For domain-specific applications, add a system message that primes the LLM with your domain knowledge:

```ballerina
function buildPrompt(string question, string context) returns string {
    return string `You are a technical support assistant for Acme Corp products.
You answer questions using only the documentation provided in the context.
Respond in a professional, concise tone.
If the documentation does not cover the topic, direct the user to support@acme.com.

Context:
${context}

Question: ${question}

Answer:`;
}
```

## Error handling

Handle common failure scenarios gracefully:

```ballerina
resource function post query(QueryRequest request) returns QueryResponse|http:InternalServerError {
    do {
        // ... main logic ...
        return {answer, sources};
    } on fail error e {
        log:printError("Query failed", 'error = e);
        return <http:InternalServerError>{
            body: {message: "Failed to process your query. Please try again."}
        };
    }
}
```

:::warning
Never expose raw error details to clients. Log the full error server-side and return a user-friendly message. For production error handling patterns, see [Error Handling](/docs/develop/build/error-handling).
:::

## What's next

- [Choose Chunking and Embedding Strategies](chunking-embedding.md) -- Optimize retrieval quality by tuning how documents are split and embedded
- [RAG Architecture Overview](architecture-overview.md) -- Review the full RAG pattern and design decisions
- [Vector Database Configuration](vector-database.md) -- Set up persistent vector stores for production
- [AI-Powered Customer Support Agent](/docs/genai/tutorials/customer-support-agent) -- Build a full agent that uses RAG for FAQ retrieval
