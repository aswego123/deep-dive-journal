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

# Quantization

Parent: [[AI Inferencing]]

## 🪝 The Hook
A 70-billion-parameter model, stored in normal precision, weighs about 140GB. That won't fit in a single GPU. But a "quantized" version of the *same model* can be 40GB or even 20GB — and it runs 2–4× faster with **barely noticeable quality loss**. That sounds like cheating. It kind of is. Here's how the trick works.

## ⚙️ Core Mechanism

Every parameter (dial) in the model is a number. During training, those numbers are stored in **32-bit floating point** (FP32) or **16-bit** (FP16 / BF16) — meaning each number takes 4 or 2 bytes of memory.

Quantization says: "**Do we really need that much precision to *use* the model?**" Turns out — no.

You can round each parameter to fit in:
- **8-bit** (INT8) — 1 byte each. Half the memory of FP16.
- **4-bit** (INT4) — 4 bits each. Quarter the memory of FP16.
- Even **2-bit** or **1.58-bit** ("ternary") in experimental cases.

So a 70B model:
- FP16: 140 GB
- INT8: 70 GB
- INT4: 35 GB

Suddenly the model fits on much cheaper hardware, uses much less memory bandwidth (which as we know from [[Prefill and Decode]] is the actual decode bottleneck), and runs a lot faster.

### Why doesn't this destroy the model?

Because neural networks are surprisingly *robust to rounding noise*. Most parameters are already tiny numbers — the difference between `0.0234` and `0.0230` doesn't change what the model does. What matters is the *rough shape* of the numbers, not their exact digits.

But it's not zero cost. Blindly rounding *everything* to 4 bits will hurt quality. So real quantization schemes are smarter:

- **Weight-only quantization** — round the *stored* parameters, but do the actual math in higher precision after un-rounding. Best quality, decent speed win.
- **Weight + activation quantization** — round the intermediate math results too. Bigger speed win, quality drops more.
- **Group-wise / per-channel scales** — apply different rounding "zoom levels" to different chunks of the model to preserve the numbers that matter most.
- **GPTQ, AWQ, SmoothQuant** — famous methods that carefully choose which numbers to round harder and which to keep precise, minimizing damage.
- **QAT (Quantization-Aware Training)** — retrain the model briefly with rounding *in the loop* so it *learns* to be robust. Best quality, most expensive.
- **PTQ (Post-Training Quantization)** — round after training with no retraining. Cheap, usually good enough.

Rough intuition of the quality tax:

- FP16 → INT8: essentially free, quality drop is usually below noise
- FP16 → INT4: small but real quality drop, often acceptable
- FP16 → INT2: noticeable quality drop, only worth it in extreme constraints (phones)

### Why it matters more than it sounds

Quantization isn't just "save disk space." It's the main lever for:

- **Running big models on small hardware** — Llama-70B on a single consumer GPU only exists because of 4-bit quantization
- **Running any model on phones** — INT4 or INT8 is basically mandatory for on-device
- **Making decode faster** — since decode is memory-bandwidth-bound, halving the size of every number literally doubles decode speed
- **Fitting more users per GPU** — smaller model means more room for KV caches, means bigger batches, means lower cost per token

Kitchen analogy: the chef's recipe book was originally written in dense calligraphy that took up 200 pages. Someone re-copied it in shorthand — 50 pages, same recipes, chef can flip through way faster. A few footnotes got fuzzy but nothing important was lost.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Prefill and Decode]] — quantization helps decode a *lot* because decode is memory-bound
- [[Inference Hardware]] — modern GPUs have dedicated INT8/INT4 math units that only exist because of quantization
- [[Cost of Inference]] — quantization is one of the top 3 cost-reduction levers
- [[Where Inference Runs - Cloud vs Edge]] — edge inference basically doesn't exist without quantization

## 💡 So What
When picking or downloading a model, always check what precision it's in. A "70B model" in FP16 needs an $80,000 GPU; the *same* model in INT4 runs on a $2,000 gaming rig. That is the single biggest price/access lever in open-source AI right now.

## ❓ Open Question
The BitNet paper (1.58-bit weights) claims you can quantize *at training time* and get similar quality to full precision with dramatically lower inference cost. If that pans out, does the whole conventional "train in FP16, quantize after" pipeline go away? Need to track how BitNet scales.

## 📚 Source
- "GPTQ" (Frantar et al., 2022)
- "AWQ" (Lin et al., 2023)
- "The Era of 1-bit LLMs / BitNet b1.58" (Ma et al., 2024)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
