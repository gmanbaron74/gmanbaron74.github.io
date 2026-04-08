---
layout: post
title: "Running Local AI on a Home Server: Getting Started with Ollama"
date: 2026-04-08
categories: [ai, home-server]
---

If you've ever wanted to run your own AI assistant without paying API fees or 
sending your data to the cloud, this guide is for you.

## What You Need

- A PC with a decent GPU (RTX 3060 or better recommended)
- A home server or spare machine for the gateway
- About an hour of setup time

## Why Run Local AI?

Running AI locally means your conversations stay private, there are no usage 
limits, and once set up the ongoing cost is essentially zero.

Popular models like Gemma, Qwen, and DeepSeek now run well on consumer hardware.
A mid-range GPU like an RTX 4070 can handle 8-14 billion parameter models 
comfortably.

## Getting Started with Ollama

Ollama is the easiest way to run local AI models. Download it from 
[ollama.ai](https://ollama.ai) and install it on your GPU machine.

Then pull a model:

```bash
ollama pull gemma4:e4b
```

That's it — you now have a local AI model running on your own hardware.

## Next Steps

In future posts I'll cover connecting Ollama to OpenClaw for a full 
Telegram-based AI assistant, setting up automatic model switching, 
and getting a morning briefing delivered to your phone every day.
