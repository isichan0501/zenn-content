---
title: "Thinking Machines releases Inkling 2 Mini, an open model for laptops"
emoji: "🤖"
type: "tech"
topics: ["ai", "open-weights", "local-llm", "inference", "reasoning"]
published: true
---

[November 21, 2026] Thinking Machines Lab, the startup led by former OpenAI CTO Mira Murati, released Inkling 2 Mini, a 26.4 billion parameter open-weights model designed to run locally on consumer laptops. The model's distinguishing architectural choice is a looped Transformer: its middle 12 of 24 layers run three times per token, enabling extended internal reasoning without generating additional text output.

According to the Artificial Analysis Intelligence Index, Inkling 2 Mini matches the performance of Qwen 2.5 30B while operating at approximately one-sixth the cost. The system has an operational memory footprint of roughly 20GB and can be run locally via Ollama on Mac systems with at least 32GB of RAM, as well as via Unsloth and llama.cpp using GGUF formats. Cloud inference, fine-tuning, and chat are available through the company's Tinker platform.

The model is released under the Apache 2.0 license, with weights available on Hugging Face and the lab's Tinker platform. Researchers can also access a companion research paper, TML-d24-varloop-research, which explores the mechanics and monitorability of recurrent transformer routing.

**Primary source**: [Thinking Machines releases Inkling 2 Mini](https://thinkingmachines.ai/news/inkling-2-mini/)

**License**: Apache 2.0. Weights on Hugging Face: `thinkingmachines/Inkling-2-Mini`.