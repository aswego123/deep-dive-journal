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

# Tokens and Embeddings

Parent: [[AI Inferencing]]

## 🪝 The Hook
The model doesn't actually read English. It has no idea what a letter is. So how does typing "hello" into ChatGPT turn into anything the model can do math on?

## ⚙️ Core Mechanism

Two steps: **tokenize**, then **embed**.

### Step 1 — Tokenize (chop the text into puzzle pieces)

The model has a fixed **vocabulary** — usually 30,000 to 200,000 predefined pieces of text. Every input has to get chopped into pieces from that vocab. Pieces are called **tokens**.

A token isn't a word, and isn't a letter. It's whatever chunks the vocab happened to memorize. Roughly:

- Common words → one token each (`" the"`, `" and"`, `" hello"`)
- Rare words → split into pieces (`"neuroplasticity"` → `"neuro" + "plastic" + "ity"`)
- Weird characters, emoji, code symbols → often their own token or split into bytes

Rule of thumb: **1 token ≈ 0.75 English words ≈ 4 characters**. So a 1,000-word essay is roughly 1,300 tokens.

The tokenizer just does a lookup: each piece → an integer ID. `"hello"` might become the number `15,043`. That's it. Now your text is a list of integers, e.g., `[15043, 220, 995, 13]`.

Why do it this way and not just letter-by-letter? Because letters would make sequences 4× longer (more math, slower), and full words would need a vocabulary of millions (too big). Sub-word tokens are the compromise.

### Step 2 — Embed (turn each integer into a "meaning vector")

Integer IDs are still meaningless to the model — `15,043` isn't more or less than `15,042`, they're just labels. So the model looks each ID up in a giant table called the **embedding matrix**.

The embedding matrix is basically:

> "For every possible token in my vocab, here is a list of ~4,000 numbers that represents what this token *means* to me."

So `"hello"` (ID 15,043) becomes a vector like `[0.21, -1.4, 0.03, ..., 0.88]` with a few thousand numbers in it. That vector is the model's private, learned representation of "hello-ness."

Tokens with similar meanings end up with similar vectors — `"king"` and `"queen"` sit near each other in that number-space; `"king"` and `"pizza"` sit far apart. Nobody hand-coded this. The model *learned* it during training, purely from reading tons of text.

So the full pipeline for your prompt:

```
"hello world"
   ↓ tokenize
[15043, 1917]
   ↓ embed (look up each ID)
[[0.21, -1.4, ...],   ← vector for "hello"
 [0.55,  0.9, ...]]   ← vector for " world"
   ↓ now the model can do math on it
```

That matrix of vectors is what actually enters the model.

## 🔗 Connects To
- [[AI Inferencing]]
- [[The Forward Pass]] — what happens *after* embedding
- [[Context Window]] — measured in tokens, not words, for exactly this reason

## 💡 So What
When a provider charges "$3 per million tokens," they're not being cute — they literally can't count in words, because the model doesn't know words. Also: languages with lots of rare characters (Japanese, code, math) burn tokens *way* faster than English, so the same paragraph can cost 2–3× more.

## ❓ Open Question
How much of a model's "vibe" (formal vs casual, etc.) is baked into the tokenizer's choices vs the training data? If two models share a tokenizer, do they end up subtly more alike?

## 📚 Source
- Concept: Byte Pair Encoding (BPE), used in most modern LLMs
- To read: Karpathy's "Let's build the GPT tokenizer" walkthrough

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
