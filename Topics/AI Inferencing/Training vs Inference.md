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

# Training vs Inference

Parent: [[AI Inferencing]]

## 🪝 The Hook
People say "AI is expensive." But *which* AI? Because training a giant model costs hundreds of millions of dollars **once**, while inference costs a few pennies **per question — a billion times a day**. These are two completely different jobs with completely different bottlenecks, and confusing them is the #1 reason AI news is confusing.

## ⚙️ Core Mechanism

Sticking with the restaurant analogy:

- **Training = culinary school.** You spend months (and a fortune) drilling the chef on thousands of recipes. Every time the chef messes up, you correct them. Slowly, the chef's instincts (the model's "dials" / parameters) get tuned. This happens **once**. It is *slow, expensive, and one-shot*.
- **Inference = the chef working at a restaurant every night.** No more learning. No more corrections. Just: order comes in → chef cooks → plate goes out. Repeat forever. This happens **billions of times**. It is *fast, cheap-per-order, but adds up*.

Here's what actually changes between the two:

| | Training | Inference |
|---|---|---|
| What's happening | Model *changes its own dials* to get better | Model *keeps its dials frozen* and just uses them |
| Direction of math | Forward pass + **backward pass** (learning) | Forward pass **only** (using) |
| Memory needed | Huge — you have to remember every step to correct it | Much smaller — just enough to answer one question |
| Speed goal | "Finish this training run in 3 months" | "Answer this user in under 1 second" |
| Runs on | Thousands of chips glued together, for weeks | Often just one chip (or even your phone), for milliseconds |
| Who pays | The lab, once | The lab (or you), every single request |

The key technical thing: **training does a forward pass AND a backward pass**. Forward = "given this input, what's my guess?" Backward = "how wrong was I, and how should I nudge every single dial to be less wrong?" That backward pass is what makes training ~3× more expensive per step than inference, *and* what makes it need way more memory (you have to hold onto intermediate results so you can trace the error backward).

Inference throws all that away. It's forward-only. That's why the same model that took 10,000 GPUs to train can often *run* on 1 GPU (or 8 GPUs for the really big ones).

## 🔗 Connects To
- [[AI Inferencing]]
- [[What is Compute]] — the "6 × params × tokens" formula is the *training* cost; inference cost is a totally different formula
- [[Scaling Laws]] — those laws are about training; there are separate scaling laws for inference cost

## 💡 So What
When I read "GPT-5 cost $2B" — that's training, one-time. When I read "OpenAI's inference bill is $700M/month" — that's the restaurant, ongoing. The second number is what actually decides whether an AI company survives.

## ❓ Open Question
For most modern AI companies, do they spend more total money on training or on inference over a model's lifetime? Rumor is inference has overtaken training as the bigger cost — need to verify.

## 📚 Source
- Companion note to [[What is Compute]]
- To read: OpenAI blog posts on inference cost trends

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
