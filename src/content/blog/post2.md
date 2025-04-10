---
title: "Building a Snappy, Reliable AI Chat with Convex"
description: "How I built a persistent, reliable chat system with Convex and the AI SDK"
pubDate: "Apr 10 2025"
heroImage: "/monsterlabs.webp"
tags: ["AI", "Chat", "Convex"]
---
# Building Persistent AI Chat with Convex: A Technical Deep Dive

*How I solved the challenges of streaming AI responses in a production app*

---

## The Goal

I wanted to build an AI chat experience that felt smooth, real-time, and persistent. I wanted to be able to swap between multiple LLM providers (OpenAI, Anthropic, Google) and support **structured object generation** and all the other cool stuff the Vercel AI SDK can do.

### Requirements:

- Assistant responses should **stream**
- **Persist** to the database as they generate
- Survive **refreshes, disconnects**, or client interruptions
- Support **multi-model** flexibility (OpenAI, Claude, Gemini…)
- Support **structured object generation** and all the other cool stuff the Vercel AI SDK can do
- Be **clean, reactive, and production-grade**

---

## Why I'm Sticking With The Vercel AI SDK (Even When It's Annoying)

I chose the [Vercel AI SDK](https://sdk.vercel.ai) because I genuinely think it's a **fantastic abstraction** for:

- Swapping between multiple LLM providers (OpenAI, Anthropic, Google)
- Supporting **structured object generation**
- Giving you a clean, unified interface to talk to models

That part? Chef's kiss.

But the `useChat()` hook?
I personally don’t think it’s the right fit for serious chatbot UIs — especially ones that need **custom control over how responses are triggered, persisted, or streamed**.

It’s fine for demos. But once you step into production, you hit walls fast.

---

## 🧪 The F5 Test

> What happens when someone hits refresh while the AI is talking? Or closes the tab?

I tested a bunch of popular AI chats. Here's what I found:

| Platform | Result | Details |
|----------|---------|----------|

| [chatgpt.com](https://chatgpt.com) | ⚠️ Partially Passes | Requires another refresh to see the complete response |
| [claude.ai](https://claude.ai/) | ❌ Fails | Response is lost entirely when the page is refreshed |
| [T3.chat](https://t3.chat) | ❌ Fails | Response is stuck at the point of interruption |
| [chat.vercel.ai](https://chat.vercel.ai) | ❌ Fails | Response is lost entirely when refreshing the page |
| [chatprd.ai](https://chatprd.ai) | ❌ Fails | Shows error alert for in-progress response, then loses it |
| [Perplexity](https://perplexity.ai) | ⚠️ Partially Passes | Requires another refresh to see the complete response |
| [sdk.vercel.ai/playground](https://sdk.vercel.ai/playground) | ❌ Fails | Chat is lost entirely on refresh |

### The Core Problem

Most of these sites stream over HTTP, which means:

I don't claim to know how all of these sites work under the hood but I suspect that many stream LLM output **over HTTP**, which means:

- The **stream is tied to the HTTP request**
- If that request is interrupted — refresh, tab close, slow network — **the response dies**
- You lose not just the connection, but the **entire message** unless you explicitly persist every chunk
- Saving the response to DB only happens when the response is complete

So, **streaming and persistence are in tension**. You get one easily. Both? Not so much.

---

## Why This Matters

If you want:

- Real-time token-by-token responses
- Messages that don't disappear
- Everything working after a refresh

Then you need an architecture that **decouples the LLM stream from the HTTP request** and **persists every chunk manually**.

Most frameworks don’t help you do this.  
The Vercel AI SDK, for example, **requires Node to run**, and assumes you're working inside **Next.js API routes**. If you’re using Convex or another edge-based backend, this isn’t ideal — you’re now streaming **across systems**, which introduces latency and risk.

---

## Why Convex is Awesome Here

I'm using [Convex](https://convex.dev) as my backend, and it's been a game-changer.

- **Reactive frontend via `useQuery`**
- **Built-in database**
- **Node-compatible `"use node"` actions**
- Extremely easy to work with and understand

With Convex, you can put your LLM logic **right next to your data** — no API hops, no serialization, no duplication. This is *exactly* what you want for performance and correctness.

---

## The Problem With Saving Every Token

If you're using `streamText()` and saving each token chunk to Convex inside `onChunk`, you’re doing a **ton of writes** — one per token.

That will hammmer your DB with writes, make your app feel slow, and cost you a lot of money.

---

## What About Queues?

I thought about using a **job queue**: generate the whole message in the background, then save it when it's done.

That works, and it's refresh-safe - but then you lose that nice streaming feel. Users have to wait for the whole thing to finish before seeing anything.

---

## My Final Architecture

Here’s what I ended up with — and it is surprisingly simple. 

### 1. **Start a the chat with `startChatMessagePair`**

This is a Convex `action` that:

- Stores the user’s message
- Creates a **blank** assistant message
- Immediately schedules a job via `ctx.scheduler.runAfter(0, ...)` to generate the assistant’s reply

This keeps the generation separate from any client request. Refresh or disconnect? No problem. The AI is still working.

---

### 2. **Generate the assistant response in `internal.llm.generateAssistantMessage`**

This is an internal Convex `"use node"` action that:

- Uses `streamText()` from the [Vercel AI SDK](https://sdk.vercel.ai) (with any model you want)
- Streams tokens with `for await...of result.textStream`
- **Keeps them in memory**
- **Saves to Convex every `150ms`** (adjust this as needed)

```ts
let buffer = "";
let accumulated = "";
let flushTimeout = null;

const flush = async () => {
  if (!buffer) return;
  accumulated += buffer;
  buffer = "";
  await ctx.runMutation(api.messages.update, {
    messageId,
    threadId,
    role: "assistant",
    content: accumulated,
    isComplete: false,
  });
};

for await (const chunk of result.textStream) {
  buffer += chunk;

  if (!flushTimeout) {
    flushTimeout = setTimeout(async () => {
      await flush();
      flushTimeout = null;
    }, 150); // Flush every 150ms. You can choose your own interval.
  }
}

if (flushTimeout) clearTimeout(flushTimeout);
await flush();

await ctx.runMutation(api.messages.update, {
  messageId,
  threadId,
  role: "assistant",
  content: accumulated,
  isComplete: true,
});
```
### Results

- Real-time feel  
- Controlled DB load  
- Persistent updates  
- Refresh safety  

No API routes. No hacky fetch calls. No streaming limits.

## What This is For: Veilborn

This whole system is part of something bigger I'm building.

Veilborn is my current obsession:

- An infinite, AI-enhanced, text-based RPG for the web.

Think:

- The story depth of The Witcher 3
- The character building of Soulslikes
- The turn-based combat of classic RPGs
- A persistent, node-based world
- Dynamic items, NPCs, and AI-driven storytelling

It's ambitious - but I've never been more excited about a project.

Also, if you're into TTRPGs, check out my side project: [MonsterLabs.app](https://monsterlabs.app), where you can create AI-driven monsters and items for D&D.

## Thanks for Reading

Hope this helps if you're trying to build something similar. Like everything in engineering, this solution has tradeoffs, but for me? It works.

I got:

- Messages that don't disappear
- Real-time feel
- Clean backend
- No hacky fetch calls
- Streaming that survives refreshes
- A relatively simple implementation

Convex is awesome - and with "use node" and the right setup, you get full control over your AI, database, and user experience.

If you're building something real with AI - don't just trust the magic SDKs. Understand what's happening under the hood. Own your infrastructure.