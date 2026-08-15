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

# Latency - TTFT and TPOT

Parent: [[AI Inferencing]]

## 🪝 The Hook
"This chat feels slow" is a useless bug report. Slow *how*? Slow to start replying? Slow while typing? Those are two totally different problems with two totally different fixes. Real inference teams don't measure "speed" — they measure two very specific numbers, TTFT and TPOT, because those are the two speeds a user actually feels.

## ⚙️ Core Mechanism

### TTFT — Time To First Token

The delay between you hitting Enter and the first word appearing.

Made up of:
- **Network round-trip** to the AI provider
- **Queue time** waiting for a GPU slot
- **Prefill** — pushing your entire prompt through the model (see [[Prefill and Decode]])
- **Sampling** the first token

For a short prompt, TTFT can be well under 300ms. For a 100k-token prompt on a big model, TTFT can be **several seconds** just for prefill alone. This is why apps that stuff huge system prompts + long chat history start to feel sluggish before the model has even "said anything."

**How to reduce TTFT:**
- Shorter prompts
- Better hardware (faster memory + more parallel math)
- Prompt caching (reuse KV cache from previous requests with the same prefix — huge win for system-prompt-heavy apps)
- Chunked prefill (start decoding before all of prefill finishes)
- Smaller model
- Less queue time (more GPUs / smaller batches)

### TPOT — Time Per Output Token

Once the reply starts streaming, the time between each subsequent word appearing. Sometimes also called "inter-token latency" (ITL).

TPOT × (number of output tokens) ≈ time to finish the full reply after TTFT.

Made up of:
- One [[The Forward Pass]] per token
- Which for decode is bottlenecked by memory bandwidth (see [[Prefill and Decode]])

Typical TPOT is 10–50ms per token depending on model size and hardware. Very roughly:
- 50 tokens/sec = 20ms TPOT — feels smooth, "reading speed"
- 20 tokens/sec = 50ms TPOT — noticeably slow
- 5 tokens/sec = 200ms TPOT — feels broken

**How to reduce TPOT:**
- Smaller model / quantization (see [[Quantization]])
- Faster memory hardware (H100 > A100 mostly because of memory bandwidth, not raw math)
- Speculative decoding (see [[Speculative Decoding]])
- Better attention kernels (FlashAttention etc.)
- Smaller batch size (each user gets more of the GPU) — but this hurts throughput

### The full picture

Full reply time ≈ **TTFT + TPOT × output_length**

So which one matters more?

- **Short replies** (like classification, extraction, small Q&A): TTFT dominates. Optimize the front end.
- **Long replies** (essays, code, reasoning chains): TPOT dominates. Optimize the streaming loop.
- **Long-prompt short-reply RAG apps**: TTFT is your whole problem. Cache aggressively.
- **Short-prompt long-reply creative writing**: TPOT is everything. Consider a smaller/faster model.

### The trade-off with throughput

Recall from [[Batching and Throughput]] that bigger batches = better throughput but worse latency. Specifically:

- Bigger batch → TPOT gets **worse** (each step processes more, takes longer)
- Bigger batch → TTFT gets **worse** (more queue time before your prefill runs)
- Bigger batch → **throughput** (tokens/sec across all users) gets much better

So a provider's SLA is basically a promise about TTFT and TPOT — and that promise is a *ceiling* on how big they'll let batches get. Which is a *floor* on their cost per token. It's all one big triangle: **speed, cost, quality — pick two**, then batching moves you along the speed↔cost edge.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Prefill and Decode]] — TTFT is prefill; TPOT is decode
- [[Batching and Throughput]] — the direct trade-off
- [[Cost of Inference]] — TTFT/TPOT SLAs directly set cost floors

## 💡 So What
When judging any AI product, watch both numbers separately. If a demo shows "instant" replies to short prompts but you're planning to feed it long documents, that demo is meaningless — you're going to hit the *other* bottleneck.

## ❓ Open Question
For interactive voice AI (e.g. real-time voice conversations), the perceptual budget is *brutal* — humans notice ~200ms of round-trip delay. How are voice AI companies getting under that budget end-to-end, including transcription and TTS? Need to look at Whisper streaming + speculative decoding combos.

## 📚 Source
- Companion to [[Prefill and Decode]] and [[Batching and Throughput]]
- Anthropic, OpenAI SLA / rate limit docs — read the definitions carefully

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
