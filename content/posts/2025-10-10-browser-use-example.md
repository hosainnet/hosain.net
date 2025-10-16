+++
date = '2025-10-10T22:55:06+04:00'
draft = false
title = 'Web browser automation with browser-use and Gemini'
+++

I recently experimented with the `browser-use` library to create an AI agent that can browse the web autonomously. This post walks through a simple example that uses Google's Gemini AI to find the top post on Hacker News's "Show HN" section.

You can find the [complete project code on GitHub](https://github.com/hosainnet/ai-experiments/tree/main/browser-use-example).

## The Setup

The implementation is fairly straightforward. Here's what we need:

- Python 3.11+ (specified in `.python-version`)
- The `browser-use` library for web automation
- Google's Gemini API for the AI model
- A few standard libraries for async operations and environment management

## The Core Implementation

The heart of the example is in `main.py`, which demonstrates the browser-use pattern in just a few lines:

```python
import asyncio
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from browser_use import Agent

async def main():
    load_dotenv()

    llm = ChatGoogleGenerativeAI(
        model="gemini-1.5-flash",
        api_key=os.getenv("GEMINI_API_KEY")
    )

    agent = Agent(
        task="Find the number 1 post on Show HN",
        llm=llm,
    )

    result = await agent.run()
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

## The AI Agent in Action

When you run this script, the AI agent will:

1. Navigate to Hacker News
2. Locate the "Show HN" section
3. Identify the top-ranked post
4. Return information about it

The agent figures out how to accomplish this task without explicit instructions about Hacker News's structure or navigation patterns.

## Why This Matters

This example represents a shift in how we think about web automation. Instead of writing brittle scripts that break when websites change their layout, we can now describe what we want to accomplish and let AI figure out the implementation details.

The `browser-use` library abstracts away the complexity of browser automation while leveraging the reasoning capabilities of large language models. This opens up possibilities for more flexible, maintainable automation scripts.

## Getting Started

To try this yourself:

1. Clone the [repository](https://github.com/hosainnet/ai-experiments/tree/main/browser-use-example)
2. Create and activate a virtual environment:
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```
3. Copy `.env.example` to `.env` and add your Gemini API key
4. Install dependencies with `uv sync` (or your preferred Python package manager)
5. Run `python main.py`
