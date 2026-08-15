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

# Batching and Throughput

Parent: [[AI Inferencing]]

## 🪝 The Hook
One GPU costs $30,000+. If it only served one user at a time, ChatGPT would need millions of GPUs and cost you $50 per question. It doesn't — because a single GPU handles hundreds of users *simultaneously* through a trick called **batching**. This is the single biggest reason inference is affordable at all.

## ⚙️ Core Mechanism

Recall from [[Prefill and Decode]] that decode barely uses the GPU's math capacity — it's memory-bound, so most math units sit idle waiting for weights to load. That's a huge waste of an expensive chip.

Batching fixes this: instead of running one user's forward pass at a time, **stack many users' current tokens into one big matrix, and do all their forward passes simultaneously.** The weights get loaded from memory *once*, then reused across everyone's math. The idle math units finally get something to do.

Kitchen analogy: instead of the chef making one pasta bowl, waiting for it to be eaten, then making the next — 20 identical pasta orders go in one giant pan at once. Same effort loading the pan onto the stove, 20× the pasta out.

### Naive batching (the old way)

- Wait until you have 32 users' requests lined up
- Run their prefill together
- Run their decode step 1 together, decode step 2 together, and so on
- The whole batch has to finish before any user gets fully done

Problem: users finish at different times (different reply lengths). If 31 users want 20-token replies and 1 user wants a 500-token reply, the GPU sits mostly idle for the last 480 steps. Ugh.

### Continuous batching (the modern way)

This is the trick that made vLLM, TGI, TensorRT-LLM etc. so much faster than earlier systems.

- Users don't have to wait for a batch to form — **new users join the batch mid-flight**
- Users who finish their reply **drop out of the batch mid-flight**
- The batch is continuously reshaped every step, keeping the GPU packed to the brim
- Typically 2–10× more throughput than naive batching

Analogy: instead of 20 pasta orders locked in one pan for a fixed time, the chef has a running pot — as one order is done and pulled out, a new one gets dropped in. Constant flow, no idle time.

### Throughput vs latency — the eternal trade-off

Bigger batches = **more throughput** (more tokens per second across all users) **but higher latency** (each individual user's reply is slower because the GPU has more work to grind through per step).

- **Latency-focused** service (e.g. real-time chat): small batches, fewer users at once, each user gets snappy replies. Expensive per-token.
- **Throughput-focused** service (e.g. batch document processing overnight): huge batches, users wait but per-token cost is minimal.

Same GPU, same model — the cost per token can differ **10×+** just based on this tuning. This is why the same model can be $5/million tokens on one provider and $0.50/million on another. They're not lying — they're on different batching regimes.

### The KV cache complication

Every user in the batch has their own [[KV Cache]] sitting in GPU memory. So the *actual* limit on batch size is often "how many KV caches fit in this GPU's memory" — not "how much math the GPU can do." A long-context user takes up the memory of many short-context users. **This is why "give me a batch size of 256" is not a real answer — it depends entirely on how long everyone's conversations are.**

Systems like vLLM's **PagedAttention** solve this by treating the KV cache like virtual memory: split into pages, share pages between users when possible, evict old pages under pressure. Doubles or triples effective batch size on the same GPU.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Prefill and Decode]] — batching helps decode most; prefill helps less because it's already compute-heavy
- [[KV Cache]] — the actual constraint on batch size
- [[Latency - TTFT and TPOT]] — batching *raises* both, on purpose
- [[Cost of Inference]] — batching is the #1 lever on cost-per-token

## 💡 So What
When comparing AI providers on price, remember: they can offer 10× lower prices by cranking batch sizes way up — at the cost of *your* individual latency. The "cheap" API and the "premium low-latency" API can be running literally the same model on the same hardware, just with different batching settings.

## ❓ Open Question
For very heterogeneous workloads (some users doing 1M-token prefill, others doing tiny decodes), continuous batching still has awkward edges. What's the state of the art on "disaggregated" serving where prefill and decode run on separate GPUs? Need to read the Splitwise/DistServe papers.

## 📚 Source
- "Orca: A Distributed Serving System for Transformer-Based Generative Models" (Yu et al., 2022) — continuous batching origin
- "vLLM" / "PagedAttention" (Kwon et al., 2023)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
