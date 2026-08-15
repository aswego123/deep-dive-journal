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

# Speculative Decoding

Parent: [[AI Inferencing]]

## 🪝 The Hook
The big model can only generate one token at a time — that's the fundamental decode bottleneck. But what if a *small, fast* model could *guess* the next 5 tokens, and the big model just had to check them all in one shot? If most guesses are right, you get a huge speedup for free — with **exactly the same output quality** as the big model would have produced. That's speculative decoding, and it's one of the wildest tricks in inference.

## ⚙️ Core Mechanism

Two models working together:
- **Draft model** — small, fast, cheap. Maybe 1B parameters. Not the smartest, but knows enough to guess plausible next words.
- **Target model** — the real one you actually want to use. Maybe 70B parameters. Slow but high quality.

The dance, per step:

1. **Draft model runs ahead** and greedily generates, say, 5 candidate tokens: `[the, cat, sat, on, the]`. This is fast — small model, sequential but cheap.
2. **Target model verifies all 5 in a single forward pass** — remember from [[Prefill and Decode]] that a forward pass can handle many tokens in parallel just fine. So verifying 5 candidates costs about the same as generating 1 the normal way.
3. For each candidate in order, compare: "would the big model have picked this token here?"
   - If yes → **accept** the token, move to the next.
   - If no → **reject** the token, replace it with what the big model *would* have said, and throw out all remaining draft tokens (they were downstream of a wrong branch).
4. If all 5 accepted → we just generated 5 tokens for the cost of ~1 big-model step. Free lunch.
5. If 3 accepted, 1 replaced, rest discarded → we got 4 tokens for the cost of ~1. Still great.
6. Repeat.

Kitchen analogy: an assistant cook throws out 5 quick guesses for the next 5 ingredients. The master chef glances at all 5 at once and either nods yes or corrects the first wrong one. If the assistant guesses well, the chef's approval covers 5 steps of work in the time it used to take for 1.

### Why the output is *identical* quality

This is the beautiful part. Speculative decoding is mathematically arranged so that the **final distribution of accepted tokens is exactly the distribution the target model would have produced on its own**. You are not "trusting the small model" — you're using the small model to *propose*, and the big model still has final say on every token. If the draft is bad, you burn a bit of extra compute rejecting; if the draft is good, you save a lot. **Quality is never lower than pure target-model generation.**

(The math: the acceptance rule uses probability ratios that ensure statistical equivalence to the target's own sampling. This is called "rejection sampling.")

### How much speedup?

Depends on how well the draft model agrees with the target:

- Easy text (formulaic, boilerplate, code with lots of repetition): 3–5× speedup common
- Hard, creative text: 1.5–2× speedup, or even nothing if drafts get rejected a lot
- Very hard text: could go *slower* than not using speculation at all (rare in practice)

Modern variants push this further:

- **Medusa** — instead of a separate draft model, add extra "prediction heads" onto the target model itself that predict tokens N+2, N+3, etc. in parallel with N+1.
- **EAGLE** — a smarter draft that operates on the target's internal features rather than raw tokens, achieving higher acceptance rates.
- **Tree-based speculation** — draft *multiple parallel guesses* per position and let the target pick the best branch. Higher acceptance overall.
- **N-gram / prompt-lookup speculation** — for tasks like code editing where the model often just *copies chunks of the prompt*, "guess by copying from the prompt" is a stupidly cheap and stupidly effective draft strategy.

### The cost side

- You now run *two* models. The small one adds real compute overhead.
- The big model does bigger forward passes (verifying multiple tokens), which costs more per step.
- If acceptance is low, you paid for both and got no benefit.
- Complicated to combine with batching — verifying different candidates across users in one batch is tricky.

Real production systems benchmark carefully — speculative decoding is not always a win. But when it is, it's a huge one, and it's the default in most modern inference stacks now.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Prefill and Decode]] — this trick specifically attacks the decode bottleneck
- [[The Forward Pass]] — exploits the fact that forward passes can verify many tokens in parallel
- [[Latency - TTFT and TPOT]] — a direct TPOT reduction trick
- [[Batching and Throughput]] — the interaction is subtle; production systems handle both

## 💡 So What
Whenever ChatGPT / Claude / Gemini suddenly feel snappier without any announced model change, one of the likely causes is a better speculative decoding scheme rolled out under the hood. The users see "faster same-quality replies" — the engineers see months of clever draft-model tuning.

## ❓ Open Question
Does speculative decoding stack cleanly with quantization and MoE, or do the interactions get weird? Especially for MoE — different tokens can activate different experts, which should mess with verification. Need to look at how mixture-of-experts models handle this.

## 📚 Source
- "Fast Inference from Transformers via Speculative Decoding" (Leviathan et al., 2023)
- "Accelerating LLM Inference with Speculative Sampling" (Chen et al., DeepMind, 2023)
- "Medusa" (Cai et al., 2024)
- "EAGLE" (Li et al., 2024)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
