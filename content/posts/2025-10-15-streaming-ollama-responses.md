+++
date = '2025-10-15T10:18:30+04:00'
draft = false
title = 'Streaming LLM responses in Ollama and Deno'
+++

When building AI-powered applications, one of the most frustrating user experiences is waiting for a complete response to load, especially for long-form content. Imagine requesting a 10,000-word story and having to wait 30+ seconds staring at a blank screen before any text appears. This is where streaming comes to the rescue.

## The Problem with Blocking Responses

Traditional AI implementations follow a request-response pattern where the entire response must be generated before being sent to the client. For short queries, this works fine. But for longer content generation, users are left wondering if the system is still working or has crashed.

## The Streaming Solution

You can find the full solution on [GitHub](https://github.com/hosainnet/ai-experiments/tree/main/deno-ollama-streaming-response).

The code in `main.ts` demonstrates a chunked response solution using Ollama with LangChain and Deno's native streaming capabilities:

```typescript
const response = await model.stream(inputText);
const stream = new ReadableStream({
    async start(controller) {
        for await (const chunk of response) {
            controller.enqueue(new TextEncoder().encode(chunk.content));
        }
        controller.close();
    }
});
```

## How It Works

1. **Ollama Streaming**: Instead of using `model.invoke()` which waits for the complete response, we use `model.stream()` which returns an async iterator of chunks.

2. **ReadableStream Creation**: We create a native web ReadableStream that processes these chunks as they arrive from the AI model.

3. **Chunk Processing**: Each chunk from Ollama is immediately encoded and enqueued to the stream, making it available to the client without waiting for the complete response.

4. **Real-Time Delivery**: The HTTP response streams these chunks directly to the client, creating a typewriter effect where text appears progressively.

Streaming AI responses transform the user experience from "is this thing broken?" to "wow, I can see it thinking and writing in real-time.", a subtle yet big difference.
