---
title: "RAG Querying"
description: "Understand the query phase of RAG -- how a question is answered using retrieved context and an LLM."
---

# RAG Querying

The query phase is where RAG delivers its value. When a user asks a question, the system retrieves relevant content from the knowledge base, augments the question with that context, and sends it to the LLM to generate a grounded answer. The LLM never sees your entire knowledge base -- it only receives the most relevant chunks for each query, which keeps costs down and responses focused.

## The Query Pipeline

The query phase follows a retrieve-then-generate flow:

```
User Question
        |
        v
    [ Embed ]      -- Convert the question into a vector
        |
        v
   [ Retrieve ]    -- Find the most similar chunks from the vector store
        |
        v
    [ Augment ]    -- Combine retrieved chunks with the original question
        |
        v
   [ Generate ]    -- Send the augmented prompt to the LLM
        |
        v
  Grounded Response
```

Each step plays a specific role in producing an accurate, contextual answer.

### Step 1: Embed the question

The user's question is converted into a vector using the same embedding model that was used during ingestion. This ensures that the question vector lives in the same semantic space as the stored document vectors, making similarity comparison meaningful.

### Step 2: Retrieve relevant chunks

The question vector is compared against all stored vectors in the vector database using similarity search (typically cosine similarity). The system returns the top-K most similar chunks -- the ones whose content is most semantically related to the question.

The value of K (how many chunks to retrieve) is a trade-off:

| K value | Effect |
|---|---|
| **Small (3-5)** | Focused context, lower token costs, risk of missing relevant information |
| **Medium (5-10)** | Good balance for most use cases |
| **Large (10-20)** | Comprehensive context, higher token costs, may include less relevant chunks |

You can also apply **filters** to narrow retrieval by metadata -- for example, only retrieving chunks from a specific document category, department, or date range.

An optional **reranking** step can improve quality by using a secondary model to re-score the retrieved chunks based on their relevance to the specific question, rather than relying solely on vector similarity.

### Step 3: Augment the question

The retrieved chunks are combined with the original question into a single prompt. This augmented prompt gives the LLM both the user's question and the specific context it needs to answer accurately. A typical augmented prompt looks like:

```
Context:
[Retrieved chunk 1]
[Retrieved chunk 2]
[Retrieved chunk 3]

Question: How many annual leave days do employees get?

Answer based on the context provided above.
```

The augmentation step is where RAG fundamentally differs from a plain LLM call. Instead of relying on the model's training data, you supply the exact information needed to answer the question.

### Step 4: Generate the response

The augmented prompt is sent to the LLM, which generates a response grounded in the provided context. Because the LLM has the relevant chunks right in front of it, the answer is specific to your data rather than generic training knowledge.

## Query Pipeline in Ballerina

Here is a complete example of the RAG query phase using the official `ai` module:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

public function main() returns error? {
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();
    check knowledgeBase.ingest(documents);

    string query = "How many annual leave days?";
    ai:QueryMatch[] queryMatches = check knowledgeBase.retrieve(query, 10);
    ai:Chunk[] context = from ai:QueryMatch queryMatch in queryMatches select queryMatch.chunk;
    ai:ChatUserMessage augmentedQuery = ai:augmentUserQuery(context, query);
    ai:ChatAssistantMessage assistantMessage = check modelProvider->chat(augmentedQuery);
    io:println("Answer: ", assistantMessage.content);
}
```

In this example:

- `knowledgeBase.retrieve(query, 10)` handles embedding the question and performing similarity search, returning the top 10 matches.
- `ai:augmentUserQuery(context, query)` combines the retrieved chunks with the original question into an augmented prompt.
- `modelProvider->chat(augmentedQuery)` sends the augmented prompt to the LLM and returns the grounded response.

## Quality Considerations

The quality of RAG query responses depends on several factors:

- **Retrieval quality** -- If the right chunks are not retrieved, the LLM cannot produce a good answer. This is influenced by chunking strategy, embedding model choice, and the value of K.
- **Chunk relevance** -- Retrieved chunks should contain the information needed to answer the question. Reranking can improve this by filtering out marginally relevant chunks.
- **Prompt design** -- How you structure the augmented prompt affects the LLM's ability to use the context effectively. Clear instructions (e.g., "Answer based only on the provided context") reduce hallucination.
- **Model capability** -- More capable models are better at synthesizing information from multiple chunks and acknowledging when the context does not contain enough information to answer.

## What's Next

- [RAG Ingestion](rag-ingestion.md) -- How the ingestion phase prepares documents for retrieval
- [Embeddings and Vector Databases](embeddings-and-vector-databases.md) -- How embeddings and vector search work under the hood
