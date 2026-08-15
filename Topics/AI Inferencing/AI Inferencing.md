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

# 🍽️ AI Inferencing — The Big Simple Book

Parent: [[AI and Compute]]

> Training is *teaching* the chef. Inference is *running the restaurant*.
> This whole book is about the restaurant.

If [[What is Compute]] and [[Scaling Laws]] were about how a model gets *born*, this cluster is about how that model *lives its day job* — answering your questions, one token at a time, fast enough that you don't get bored waiting.

---

## 🪝 Why this deserves a whole book

Training gets all the headlines ("we spent $500M training GPT‑X"). But once a model is trained, it will answer questions **billions of times a day, for years**. Almost every dollar, every watt of power, every millisecond of "why is ChatGPT slow today" — that's inference.

If you understand inference, you understand:

- Why AI answers stream out word-by-word instead of appearing all at once
- Why long chats slow down over time
- Why "context window" is such a big deal
- Why the same model costs 100× less on your phone than in a data center (or vice versa)
- Why running AI is a *systems* problem, not just a *smart model* problem

---

## 📚 How to read this book (in order)

Read these top to bottom. Each one leans on the one before it. Don't skip.

### Part 1 — The absolute basics
1. [[Training vs Inference]] — the two totally different jobs
2. [[Tokens and Embeddings]] — how your text becomes numbers
3. [[The Forward Pass]] — what "one prediction" actually is

### Part 2 — How a real chat actually works
4. [[Prefill and Decode]] — the two phases every reply goes through
5. [[KV Cache]] — the memory trick that makes chat possible
6. [[Context Window]] — why the model "forgets" past a certain point

### Part 3 — Serving many people at once
7. [[Batching and Throughput]] — squeezing more replies out of one GPU
8. [[Latency - TTFT and TPOT]] — the two speeds users actually feel

### Part 4 — Making it cheaper and faster
9. [[Quantization]] — shrinking the model without breaking it
10. [[Speculative Decoding]] — the "guess and check" speed hack
11. [[Inference Hardware]] — GPUs, TPUs, NPUs, CPUs — who does what

### Part 5 — The bigger picture
12. [[Cost of Inference]] — where the money actually goes
13. [[Where Inference Runs - Cloud vs Edge]] — data center vs your laptop vs your phone

---

## ⚙️ The 30-second version of everything

- **Training** teaches the model once. **Inference** uses the model forever.
- Every reply is generated **one token at a time**. The model predicts the next token, tacks it on, feeds the whole thing back in, predicts the next one. Repeat until done.
- This has **two phases**: *prefill* (read your entire prompt fast, in parallel) and *decode* (spit out the reply one token at a time, slow).
- The **KV cache** is a scratchpad that saves the model from re-reading the whole conversation every single time it produces a new word. Without it, chat would be unusably slow.
- **Batching** = one GPU serves many users at once by stacking their requests together, like a chef cooking 20 identical pasta orders in one big pan.
- **Latency** (how fast your reply *feels*) and **throughput** (how many replies the GPU cranks out per second) *trade off against each other*.
- **Quantization** = use smaller numbers (4-bit instead of 16-bit) so the model fits in less memory and runs faster, at a small quality cost.
- **Speculative decoding** = a small, fast model *guesses* the next few tokens; the big model just checks them. Much faster when the guesses are right.
- **Context window** = the max amount of text (prompt + reply so far) the model can hold in mind at once. Bigger = more useful, but *quadratically* more expensive.
- **Where it runs** shapes everything: cloud GPU = powerful & shared, edge/phone = private & offline & tiny.

---

## 🔗 Connects To
- [[AI and Compute]] — the parent cluster
- [[What is Compute]] — the math foundation this book assumes
- [[Scaling Laws]] — why models got so big in the first place
- [[Parallelism]] — training's version of "many chefs"; inference has its own version

## 💡 So What
Once I've read this whole book, I should be able to look at any AI product and roughly guess: *is this bottlenecked by prefill or decode? Is it running on-device or in the cloud? Is it batched? Is it quantized?* Those four questions basically explain 90% of why any AI product feels the way it feels.

## ❓ Open Question (for the whole book)
Where is the actual ceiling on inference cost reduction? Every year quantization + speculative decoding + better hardware make it cheaper — but at some point physics wins. When does that curve flatten out?

## 📚 Sources
- Companion to [[What is Compute]], [[Scaling Laws]], [[Parallelism]]
- Foundational papers to read as I go: "Attention Is All You Need" (2017), "Efficiently Scaling Transformer Inference" (Google 2022), "vLLM / PagedAttention" (2023), "Speculative Decoding" (Leviathan et al. 2023)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
