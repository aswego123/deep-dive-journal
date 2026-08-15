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

# The Forward Pass

Parent: [[AI Inferencing]]

## 🪝 The Hook
When ChatGPT streams a reply word-by-word, what is actually happening between each word appearing on your screen? The answer: a **forward pass** — one full trip through the entire model, billions of multiplications, just to pick the next word. And then it does the whole thing again for the word after that. And again. And again.

## ⚙️ Core Mechanism

A forward pass is: **take the current input, push it through every layer of the model, and out the other end pops a guess for what comes next.**

Concretely, for a modern LLM (transformer), those "layers" are stacked identical blocks — a small model might have 24 of them, a big one might have 80+. Each block does the same recipe:

1. **Attention** — every token in the sequence "looks at" every other token and decides who's relevant. This is how the model figures out that in "the cat sat on the mat, it was fluffy," the word "it" refers to "cat" and not "mat." Attention is the magic ingredient — it's how the model handles *context*.
2. **Feed-forward network** — after attention decided *what* to focus on, this part does the actual "thinking" on those focused-on values. Basically a giant lookup / transformation.
3. **Residual + normalize** — keep the signal from getting scrambled as it passes through 80 layers.

Do that 80 times in a row. What comes out the other end is a giant list of numbers — one number for every single token in the entire vocabulary (all 100,000+ of them). Those numbers are called **logits**, and they're basically the model's "score" for how likely each possible next token is.

Then a tiny final step called **sampling** picks the winner:

- **Greedy** — just pick the highest scorer every time. Boring but predictable.
- **Temperature sampling** — pick randomly, but weight higher-scoring tokens more. Higher "temperature" = more chaotic/creative. Lower = more repetitive/safe.
- **Top-k / top-p** — only sample from the top few candidates, ignore the tail. Prevents the model from picking weird nonsense words.

Whatever token gets picked becomes the next word on your screen — and immediately gets appended to the input, and the **entire forward pass runs again** to pick the token after that.

So: one forward pass = one token. A 500-token reply = 500 forward passes = 500 full trips through every layer of the model. This is why long replies take longer — not because the model is "thinking harder," but because it's literally doing 500 separate rounds of the same math.

The cost of *one* forward pass is roughly **2 × parameters** worth of math (measured in FLOPs). So for a 70-billion-parameter model, generating one token = about 140 billion multiplications. And you do that 500 times for a paragraph.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Tokens and Embeddings]] — the input side of the pass
- [[Prefill and Decode]] — the two *modes* of running forward passes
- [[What is Compute]] — training was "6 × params × tokens"; inference per-token is "2 × params" — the other 4× was the backward pass we no longer need

## 💡 So What
"The model is generating…" means "the model is doing another billion-multiplication trip through itself, just to pick one word." That's why word-by-word streaming is a UX choice made necessary by physics — you literally cannot get the whole answer before the last forward pass finishes.

## ❓ Open Question
Every forward pass currently activates *all* the model's parameters. But "Mixture of Experts" models only activate a fraction. Does that break the "2 × params per token" rule? I think it does, but I'm not sure of the exact accounting. Need to dig into MoE.

## 📚 Source
- "Attention Is All You Need" (Vaswani et al., 2017) — the transformer paper
- Companion to [[What is Compute]]

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
