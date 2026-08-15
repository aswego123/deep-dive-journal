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

# Context Window

Parent: [[AI Inferencing]]

## 🪝 The Hook
When people brag "our model has a 1 million token context window!" — that number sounds impressive, but almost nobody explains *why bigger is hard* and *what it costs*. Turns out attention (the magic ingredient of transformers) scales **quadratically** with context length. Double the context, quadruple the cost. This one fact shapes almost every long-context AI product you use.

## ⚙️ Core Mechanism

The context window is: **the maximum number of tokens the model can hold in mind at once** — that includes your system prompt, your entire conversation history, any attached documents, AND the reply it's currently generating. When you hit the limit, older tokens get dropped or the model refuses.

Common sizes today:
- Small: 4k – 8k tokens (~3–6k words)
- Medium: 32k – 128k tokens (a whole book chapter)
- Large: 200k – 1M+ tokens (a whole novel, or a whole codebase)

### Why bigger is *quadratically* harder

Attention (from [[The Forward Pass]]) is "every token looks at every other token." If you have N tokens, that's N × N = N² comparisons per attention layer.

- 1,000 tokens → 1 million comparisons
- 10,000 tokens → 100 million comparisons (100× more)
- 100,000 tokens → 10 billion comparisons (10,000× more)
- 1,000,000 tokens → 1 trillion comparisons (1,000,000× more)

That N² curve is brutal. That's why a 1M-token context isn't "10× harder than 100k" — it's ~100× harder. This is *the* central technical problem of long context.

### Why bigger is memory-hungry too

Remember [[KV Cache]] grows linearly with tokens? So a 1M-token context = a massive KV cache. For a big model, this can be tens of gigabytes *per user*. Which means:

- Long context = fewer parallel users per GPU
- Long context = more expensive per query
- Long context = slower prefill (linear in context length even before N² attention kicks in)

### How labs actually pull off million-token contexts

They cheat, cleverly. A few tricks:

- **Sparse / windowed attention** — a token only looks at nearby tokens plus a few far-away "landmarks." Approximates full attention at way lower cost.
- **Flash Attention** — same attention math, but computed in a smarter memory pattern that avoids re-reading data. Not a speedup of *math*, a speedup of *memory access*.
- **Ring / distributed attention** — split the sequence across many GPUs; each one handles a slice, they pass results in a ring.
- **Position encoding tricks (RoPE, ALiBi, YaRN)** — teach the model how to handle positions much further than it was originally trained on.
- **Retrieval instead of context** — don't stuff the whole book in; use a search index to find the 5 relevant paragraphs and stuff *those* in. Basically "cheat by not actually using a big context."

### The dirty secret: "effective context" ≠ "advertised context"

Just because a model *accepts* 1M tokens doesn't mean it *uses* them well. Multiple studies show a "**lost in the middle**" effect — models pay a lot of attention to the beginning and end of the context and skim the middle. So the *usable* context can be much smaller than the advertised limit. Always test this before trusting it.

## 🔗 Connects To
- [[AI Inferencing]]
- [[The Forward Pass]] — attention is the reason N² shows up
- [[KV Cache]] — the memory cost of long context
- [[Prefill and Decode]] — long context = long prefill
- [[Scaling Laws]] — those are about *training* data, this is about *runtime* input; different beasts

## 💡 So What
"Bigger context window" is not a free feature. It costs quadratic compute for attention, linear memory for KV cache, and often noticeable quality drops in the middle of the window. When picking a model, I should ask: what's the *effective* usable context, and how much does using it actually cost per query?

## ❓ Open Question
Sub-quadratic architectures (Mamba, RWKV, state space models) claim to fix the N² problem entirely by ditching attention. Do they actually match transformer quality yet? Every year the answer seems to be "almost." Need to check the current state.

## 📚 Source
- "Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., 2023)
- "FlashAttention" (Dao et al., 2022)
- "RoPE" and "YaRN" papers on position encoding

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
