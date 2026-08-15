---
date: 2026-08-15
topic: AI Inferencing
season: AI and Compute
tags:
  - deep-dive
  - ai
  - inference
status: seedling
---

# Prefill and Decode

Parent: [[AI Inferencing]]

## 🪝 The Hook
Why is there a delay before ChatGPT starts typing, and then once it starts, it types at a fairly steady speed? Those two phases feel different because they *are* different — they're actually two totally different computer workloads running back-to-back. They're called **prefill** and **decode**, and understanding this split explains most weird performance behavior in AI chat.

## ⚙️ Core Mechanism

Every reply happens in two phases.

### Phase 1 — Prefill ("read the whole question at once")

You send a prompt: `"Explain quantum computing in simple terms."` That's, say, 8 tokens.

The model has to read and understand all 8 tokens before it can start replying. Crucially — it reads all 8 **in parallel, in a single big forward pass**. The GPU absolutely loves this because it's built for doing thousands of multiplications simultaneously. Prefill is *massively parallel* — it hammers the GPU's raw math power.

Cost of prefill: roughly proportional to **(prompt length) × (model size)**. A long prompt takes longer to prefill.

What you feel: the pause *before* the first word appears. This is called **Time to First Token (TTFT)**. Long prompts = long TTFT.

### Phase 2 — Decode ("write the reply, one word at a time")

Now the model starts producing the reply. But here's the catch: **it can only produce one token at a time, and each token depends on the one before it.** You can't generate word #7 before word #6 exists, because word #6 is part of the input to word #7's forward pass.

So decode is *sequential* — one forward pass per token, waiting for each to finish before starting the next. The GPU hates this. It has thousands of math units sitting idle because there's only one small piece of work at a time.

What you feel: the steady stream of words appearing after the first one. Speed is called **Tokens Per Output Token time (TPOT)**, or just "tokens per second."

### The weird bottleneck: memory bandwidth

Here's the counterintuitive part. Prefill is bottlenecked by **raw math speed** (compute-bound). Decode is bottlenecked by **memory bandwidth** — how fast the GPU can *read the model's weights out of memory* — because for every single token, you have to load all 70 billion (or whatever) parameters from GPU memory to the math units, just to do a *tiny* amount of math on them and throw them away.

Kitchen analogy: prefill is like a chef prepping 100 ingredients all at once, everyone busy. Decode is like sending one waiter into a giant pantry to fetch every ingredient in the whole restaurant, just to make one small bite, then doing it all over again for the next bite. The pantry trip (memory) dominates the cooking time (math). This is why **decoding barely gets faster when you buy a "faster" GPU with more math horsepower** — the memory pipe is the bottleneck, not the math.

### Why this split matters

- Prefill and decode have **totally different performance profiles**, so modern inference systems often *schedule them separately*, sometimes even on different GPUs. This is called "disaggregated serving."
- Long prompt + short reply = prefill-dominated. Short prompt + long reply = decode-dominated. These two workloads look nothing alike from the GPU's perspective.
- Almost every inference optimization is really trying to fix one specific pain: either "make prefill faster" or "get around the memory-bandwidth wall in decode."

## 🔗 Connects To
- [[AI Inferencing]]
- [[The Forward Pass]] — both prefill and decode are made of forward passes; they just differ in how many happen at once
- [[KV Cache]] — the trick that makes decode not have to redo prefill work every step
- [[Latency - TTFT and TPOT]] — the two user-visible speeds are literally the two phases
- [[Batching and Throughput]] — batching helps decode a *lot* precisely because decode has spare GPU capacity

## 💡 So What
When someone says "our AI is fast," ask: fast at prefill (short TTFT) or fast at decode (high tokens/sec)? Those are two completely different engineering wins. And when a chat feels slow, I can now roughly guess *which* phase is the culprit based on whether the delay is before the first word or during the stream.

## ❓ Open Question
For very long context prompts (100k+ tokens), prefill can take *seconds*. Are there tricks to start decoding before prefill is fully done? I've seen the term "chunked prefill" — need to understand it.

## 📚 Source
- "Efficiently Scaling Transformer Inference" (Google, 2022)
- "Splitwise" and "DistServe" papers on disaggregated prefill/decode

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
