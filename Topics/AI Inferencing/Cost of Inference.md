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

# Cost of Inference

Parent: [[AI Inferencing]]

## 🪝 The Hook
The price of running AI has dropped **~10× per year** for several years running. Not because the hardware got 10× faster (it didn't). So where does the price drop come from? Turns out inference cost is a stack of maybe 8 different levers, each pulling a little, and they multiply. Understand the levers, understand the economics of the whole industry.

## ⚙️ Core Mechanism

### What "cost" even means

Two different meanings, don't mix them up:

- **Cost to the provider** (fully-loaded cost of running a GPU: hardware amortization + electricity + cooling + network + engineering) — usually measured in **$/GPU-hour** (~$2–$8/hr for H100 on cloud).
- **Cost to the customer** (what OpenAI/Anthropic/etc. charges) — measured in **$ per million tokens**, split into "input tokens" and "output tokens" pricing. Output is usually 3–5× more expensive than input because decode is slower per token than prefill.

The gap between these two is the provider's margin, which right now is *massive* for hosted APIs (there's a reason OpenAI's revenue is going up).

### The formula, roughly

For a provider:

> **Cost per token ≈ (GPU $/hour) / (tokens generated per hour on that GPU)**

So there are exactly two ways to reduce cost:
- **Denominator up** — squeeze more tokens per GPU-hour
- **Numerator down** — cheaper hardware or cheaper power

### The levers (denominator)

Every one of these is a full chapter of this book. All of them stack.

1. **Batching** ([[Batching and Throughput]]) — 10–100× more tokens/hour vs single-user. This is the single biggest lever, especially continuous batching.
2. **Quantization** ([[Quantization]]) — INT4 vs FP16 gives ~4× less memory used per parameter → bigger batches fit → more tokens/hour, *plus* decode speed roughly doubles.
3. **Speculative decoding** ([[Speculative Decoding]]) — 1.5–3× more tokens/second.
4. **KV cache tricks** ([[KV Cache]]) — PagedAttention, prompt caching, MQA/GQA. Each independently boosts effective batch size 2–5×.
5. **Better attention kernels** — FlashAttention v2/v3 and similar. Small percentage wins that add up.
6. **Distillation** — train a small model to mimic a big model. Smaller model = same task at a fraction of the cost. (Different from quantization: quantization compresses; distillation retrains a new model.)
7. **Mixture of Experts (MoE)** — activate only a fraction of parameters per token. A "300B MoE" might only use 40B params per token — inference cost roughly like a 40B model with the quality closer to a 300B one.
8. **Sparse / linear attention alternatives** — for very long contexts, get out from under the N² attention curve.

### The levers (numerator)

- **Cheaper hardware** — as [[Inference Hardware]] shows, matching workload to right chip matters a lot. Also newer generations (B200 vs H100) offer big perf/$ jumps.
- **Cheaper power** — data center location, energy contracts, renewable / off-peak.
- **Better utilization** — a GPU sitting idle costs the same as a fully loaded one. Scheduling matters.

### Where the money actually goes for one API call

For a typical hosted API call, a rough breakdown of the *provider's* cost:

- ~60–75%: GPU amortization
- ~15–25%: electricity + cooling
- ~5–10%: networking, storage, control plane
- ~5%: engineering / R&D allocation

Different providers' pricing tells you what they're optimizing for. A provider charging \$3/M output tokens on a big model is running low batches (latency-focused). A provider charging \$0.30/M for the same model is running high batches (throughput-focused). Both can be reasonable businesses on the same hardware.

### The scary compounding math

If you're serving:
- 1 million users
- 100 requests/day per user
- 500 output tokens per request

That's **50 billion output tokens per day**. At \$3/M tokens: **$150,000/day = ~$55M/year** just to serve one product line. This is why *inference* — not training — is the dominant cost for any mature AI product. And why every one of the levers above is worth an engineering team.

### Why price drops 10× per year

Roughly: hardware ~2×, batching improvements ~1.5×, quantization ~1.5×, speculative + KV improvements ~1.5×, MoE and architectural improvements ~1.5–2×. Multiply → 10×+. And this is likely to continue for at least a few more years before physics starts biting harder.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Batching and Throughput]] — the biggest single lever
- [[Quantization]] — the second biggest
- [[Speculative Decoding]] — the trendy one
- [[KV Cache]] — the memory constraint that drives everything
- [[Inference Hardware]] — the numerator
- [[Where Inference Runs - Cloud vs Edge]] — running on-device *offloads* the provider's cost entirely

## 💡 So What
When a startup says "our API is 10× cheaper than OpenAI," it doesn't mean they're geniuses — it means they've picked a different point on the batching/latency/quality curve. Cheap prices almost always mean slower per-user speed or a smaller/quantized model. Nothing is free; you're always paying somewhere in the triangle.

## ❓ Open Question
At what point does inference cost stop being a moat? If prices keep dropping 10×/year, the marginal cost of a query goes to fractions of a cent. Then business models built on token pricing collapse and the real question becomes "who has the best product on top of essentially-free inference."

## 📚 Source
- SemiAnalysis writeups on inference economics
- Andreessen Horowitz "The Economics of Generative AI" essays
- Public API pricing pages (comparative snapshots over time)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
