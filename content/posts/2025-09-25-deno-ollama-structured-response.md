+++
date = '2025-09-25T15:18:15+04:00'
draft = false
title = 'Deno, LangChain, Ollama, and structured outputs'
+++

When working with LLMs programmatically, it's important to receive a structured response from the model. This makes it easier to parse and use the output in your application.

In this post, we'll walk through how to define a schema using `zod` and query an Ollama model with the help of LangChain.

For simplicity, we’ll use [Deno](https://deno.com), so make sure it's installed on your system.

You can find the full code here: https://github.com/hosainnet/ai-experiments/tree/main/deno-ollama-structured-response — let’s break it down.

## Zod schema

We’ll define the following schema to capture the answer to the query, the source, and a description of that source.

```typescript
import { ChatOllama } from "npm:@langchain/ollama";

import { z } from "npm:zod";
import { RunnableSequence } from "npm:@langchain/core/runnables";
import { StructuredOutputParser } from "npm:@langchain/core/output_parsers";
import { ChatPromptTemplate } from "npm:@langchain/core/prompts";
import { BaseChatModel } from "npm:@langchain/core";

const zodSchema = z.object({
    answer: z.string().describe("answer to the user's question"),
    source: z.string().describe(
        "source used to answer the user's question, should be a website.",
    ),
    source_description: z.string().describe(
        "description of the source used to answer the user's question.",
    ),
});
```

## Structured query

We’ll use this schema to perform a structured query on the model. It’s passed via the `fromZodSchema` function.

```typescript
export const queryStructured = async (prompt: string, model: BaseChatModel) => {
    const parser = StructuredOutputParser.fromZodSchema(zodSchema);

    const chain = RunnableSequence.from([
        ChatPromptTemplate.fromTemplate(
            "Answer the users question as best as possible.\n{format_instructions}\n{question}",
        ),
        model,
        parser,
    ]);

    return await chain.invoke({
        question: prompt,
        format_instructions: parser.getFormatInstructions(),
    });
};
```

### Example

```typescript
const model = new ChatOllama({
    model: Deno.args[0],
});

const structuredResponse = await queryStructured("What is Deno?", model);
console.log(structuredResponse);
```
