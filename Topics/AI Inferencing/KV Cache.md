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

# KV Cache

Parent: [[AI Inferencing]]

## 🪝 The Hook
If the model generates one token at a time, and each new token has to "look at all previous tokens" (that's what attention does)… doesn't that mean by token #500, it has to re-read the previous 499 tokens *from scratch every single time*? Wouldn't that be insanely slow?

Yes. It would be. Except for one clever trick that basically makes modern chat possible: the **KV cache**.

## ⚙️ Core Mechanism

Recall from [[The Forward Pass]] that the magic step is **attention** — each token "looks at" every other token and decides how much to care about each one. Under the hood, attention computes two things for every past token:

- A **Key** (K) — "here's what I have to offer"
- A **Value** (V) — "here's my actual content"

Then the new token computes a **Query** (Q) — "here's what I'm looking for" — and matches it against everyone's Keys to figure out where to grab Values from.

Here's the thing: **the K and V for token #47 are exactly the same whether the model is currently generating token #48 or token #500.** Token #47 doesn't change! Its Key and Value are frozen forever the moment it was generated. So recomputing them at every step would be pure waste.

The KV cache is dead simple:

> After computing the Key and Value for each token, **save them in GPU memory**. When generating the next token, don't recompute — just look them up.

So on step 500, the model:
1. Computes K and V *only* for the new token #500 (cheap — one token's worth of work)
2. Reads all previous K's and V's from cache (memory lookup, no math)
3. Does attention using cached K's and V's + new Q
4. Predicts token #501
5. Appends token #500's new K and V to the cache
6. Repeat

Without the cache: generating token #500 would cost roughly **500× more math** than generating token #1. With the cache: token #500 costs about the same math as token #1. Chat would be economically impossible without this.

### The trade-off nobody warns you about

The cache is not free — **it eats memory**. A lot of memory. Roughly:

> KV cache size = 2 × layers × tokens × hidden_dim × bytes_per_number

For a big model with a long conversation, this can be **gigabytes per user**. And every simultaneous user needs their own KV cache in GPU memory. So:

- **More users at once** → GPU memory fills up with KV caches → can only serve N users per GPU
- **Longer conversations** → cache grows → same GPU serves fewer users
- **This is often the actual bottleneck on inference cost**, not raw math

Which is why almost every modern inference engine is really a **KV cache manager** dressed up as an AI system. Papers like **PagedAttention / vLLM** are literally about "how do we manage the KV cache like an operating system manages RAM" — with paging, sharing, eviction, all of it.

### Kitchen analogy

The chef is writing a really long story. Every new sentence should "reference" everything already written. Without the cache: chef re-reads the entire story from page 1 before writing each new sentence. Sentence 500 requires reading 499 pages just to start. With the cache: chef keeps a running index card of "important stuff from every previous sentence" on the counter, and just glances at that when writing the next sentence. Sentence 500 costs the same effort as sentence 5.

The cost: that stack of index cards keeps growing. Eventually it's covering the whole counter and there's no room to actually cook.

## 🔗 Connects To
- [[AI Inferencing]]
- [[The Forward Pass]] — this is optimizing the attention step of every pass
- [[Prefill and Decode]] — prefill *fills* the cache, decode *reads and appends* to it
- [[Context Window]] — the max cache size is basically the context window
- [[Batching and Throughput]] — batching many users = many caches competing for GPU memory

## 💡 So What
When a long chat starts getting "slower" or "more expensive per token," it's not that the model is thinking harder — it's usually that the KV cache is huge and eating memory bandwidth. Also: this is why "compact your conversation" or "start a new chat" isn't just UX advice — it literally frees GPU memory.

## ❓ Open Question
There are new tricks (Multi-Query Attention, Grouped-Query Attention) that shrink the KV cache by having multiple query heads share fewer K/V heads. How much quality do you lose? I've heard "basically none," but need to see the actual data.

## 📚 Source
- "Efficient Memory Management for Large Language Model Serving with PagedAttention" (Kwon et al., 2023) — the vLLM paper
- "Fast Transformer Decoding: One Write-Head is All You Need" (Shazeer, 2019) — MQA

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
