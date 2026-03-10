---
sidebar_position: 7
title: "Use Natural Functions"
description: Define typed function signatures and let the LLM execute the logic at runtime, described in plain language.
---

# Use Natural Functions

Natural functions let you describe logic in plain language and have the LLM execute it at runtime with full type safety. Instead of writing imperative code for tasks like categorization, summarization, or rating, you define the function signature — input types, output types, and a natural language description — and the LLM handles the implementation.

In this guide, you'll build a blog review system that uses a natural function to analyze blog content, suggest a category, and rate the post on a scale of 1 to 10.

:::tip Natural Programming
To learn more about natural programming and natural functions, see [Natural Language is Code: A hybrid approach with Natural Programming](https://blog.ballerina.io/posts/2025-04-26-introducing-natural-programming/).
:::

## How Natural Functions Work

A natural function has three parts:

1. **Input type** — A Ballerina record that defines the data the function receives
2. **Output type** — A Ballerina record that defines the structure of the response
3. **Natural language description** — Plain English that tells the LLM what to do

At runtime, WSO2 Integrator sends the input data and description to the LLM, then validates and parses the response into the output type.

```text
Input (typed) ──► LLM (natural language logic) ──► Output (typed & validated)
```

:::info
Because the LLM generates responses at runtime, natural function outputs may vary slightly across executions. Use specific, constrained output types (enums, bounded integers) to reduce variability.
:::

## Step 1: Create a new integration project

1. Click on the **BI** icon in the sidebar.
2. Click on the **Create New Integration** button.
3. Enter `BlogReviewer` as the project name.
4. Select the project directory and click on the **Select Location** button.
5. Click on the **Create New Integration** button to create the integration project.

## Step 2: Define types

You need two types: one for the blog input and one for the review output.

1. Click on the **Add Artifacts** button and select **Type** in the **Other Artifacts** section.
2. Click on **+ Add Type** to add a new type and switch to the **Import** section.
3. Enter `Blog` as the **Name**, paste the following JSON payload, and click the **Import** button.

```json
{
    "title": "Tips for Growing a Beautiful Garden",
    "content": "Spring is the perfect time to start your garden. Begin by preparing your soil with organic compost and ensure proper drainage. Choose plants suitable for your climate zone, and remember to water them regularly. Don't forget to mulch to retain moisture and prevent weeds."
}
```

4. Add another type with `Review` as the **Name** and paste the following JSON payload. Then click the **Import** button.

```json
{
    "suggestedCategory": "Gardening",
    "rating": 5
}
```

5. The types are now available in the project. `Blog` and `Review` represent the blog content and review respectively.

![Types defined in the project](/img/genai/natural-functions/types.png)

:::note
Importing from sample JSON is the fastest way to generate Ballerina record types. BI infers field names and types automatically.
:::

## Step 3: Add a natural function

1. Click on the **Add Artifact** button and select **Natural Function** under the **Other Artifacts** category.
2. Use `reviewBlog` as the name of the function. Click the **Add Parameter** button to add a parameter of type `Blog` named `blog`. Use `Review` as the return type and convert it to a nilable type using type operators. Then click **Create**.

![Create natural function](/img/genai/natural-functions/natural-function.png)

3. Click on the **Edit** button to specify the requirement in natural language (the prompt).
4. Use the following prompt and click **Save**. Note how interpolations refer to the `blog` parameter.

```text
You are an expert content reviewer for a blog site that
    categorizes posts under the following categories: "Gardening", "Sports", "Health", "Technology", "Travel"

    Your tasks are:
    1. Suggest a suitable category for the blog from exactly the specified categories.
       If there is no match, use null.

    2. Rate the blog post on a scale of 1 to 10 based on the following criteria:
    - **Relevance**: How well the content aligns with the chosen category.
    - **Depth**: The level of detail and insight in the content.
    - **Clarity**: How easy it is to read and understand.
    - **Originality**: Whether the content introduces fresh perspectives or ideas.
    - **Language Quality**: Grammar, spelling, and overall writing quality.

Here is the blog post content:

    Title: ${blog.title}
    Content: ${blog.content}
```

![Natural function view with prompt](/img/genai/natural-functions/natural-function-view.png)

:::tip
Notice the `${blog.title}` and `${blog.content}` template expressions. These inject the actual blog data into the prompt at runtime, so the LLM always receives the concrete values it needs to produce a review.
:::

## Step 4: Create an HTTP service

1. In the design view, click on the **Add Artifact** button.
2. Select **HTTP Service** under the **Integration as API** category.
3. Select the **Create and use the default HTTP listener (port: 9090)** option from the **Listeners** dropdown.
4. Select the **Design from Scratch** option as the **Service Contract** and use `/blogs` as the **Service base path**.
5. Click on the **Create** button to create the new service.

![HTTP Service created](/img/genai/natural-functions/service.png)

6. The service will have a default resource named `greeting` with the **GET** method. Click on the three dots in front of the `/blogs` service and select **Edit** from the menu.
7. Click the **Edit** button in front of the `/greeting` resource.
8. Change the resource HTTP method to **POST**.
9. Change the resource name to `review`.
10. Click on **Add Payload** and specify `blog` as the name and `Blog` as the type.
11. Change the 201 response return type to `Review`.
12. Click on the **Save** button to update the resource.

![Updated resource configuration](/img/genai/natural-functions/update-resource.png)

## Step 5: Implement the resource logic

1. Click on the `review` resource to navigate to the resource implementation designer view.
2. Hover over the arrow after start and click the **+** button to add a new action to the resource.
3. Select **Call Natural Function** from the node panel.
4. Select the `reviewBlog` function from the suggestions.
5. For the **Blog** parameter, use `blog` as the argument and click **Save**.

![Call natural function](/img/genai/natural-functions/call-np.gif)

6. Add a new node after the `reviewBlog` function call and select **Return** from the node panel.
7. Select the `review` variable from the dropdown and click **Save**.

![Add return node](/img/genai/natural-functions/add-return.png)

The resource implementation is now complete. The function `reviewBlog` is called with the `blog` content as input, and the `review` is returned as the response.

## Step 6: Configure model for natural function

1. Press `Ctrl + Shift + P` on Windows and Linux, or `Shift + Cmd + P` on a Mac, and type `>Ballerina: Configure default model for natural functions (Experimental)` to configure the default model for natural functions.

![Configure model for natural functions](/img/genai/natural-functions/configure-model.png)

## Step 7: Run the integration

:::warning Response May Vary
Since this integration involves an LLM call, the response values may not always be identical across different executions.
:::

1. Click on the **Run** button in the top-right corner to run the integration.
2. The integration will start and the service will be available at `http://localhost:9090/blogs`.
3. Click on the **Try it** button to open the embedded HTTP client.
4. Enter the blog content in the request body and click the button to send the request.

```json
{
    "title": "The Healthy Maven",
    "content": "For those who want a 360-degree approach to self-care, with advice for betterment in the workplace, home, gym, and on the go, look no further. The Healthy Maven offers recipes for every type of meal under the sun (salads, sides, soups, and more), DIY tips (you'll learn how to make your own yoga mat spray), and quick workouts. If you like where all this is going, there's a supplementary podcast run by blogger Davida with guest wellness experts."
}
```

5. The blog content is analyzed by the natural function to suggest a category and rate it based on the predefined criteria.

![Run integration and view results](/img/genai/natural-functions/run-integration.png)

## Best Practices

- **Constrain your output types.** Use specific field types and bounded ranges (e.g., an integer between 1 and 10) to reduce LLM variability.
- **Write precise prompts.** Vague instructions lead to inconsistent results. List the exact categories, scales, or formats you expect.
- **Handle errors gracefully.** Natural functions can fail if the LLM response does not match the expected type. Always account for the nilable return.
- **Pick the right model.** Simpler tasks (categorization) work well with faster, cheaper models. Complex tasks benefit from more capable models.

## What's Next

- [Model Selection Guide](../llm-connectivity/model-selection) — Use direct LLM calls with model providers for similar type-safe results
- [Create a Chat Agent](../agents/chat-agents) — Build conversational agents that can use natural functions as tools
- [Prompt Engineering for Integrations](../llm-connectivity/prompt-engineering) — Design better prompts for more reliable outputs
