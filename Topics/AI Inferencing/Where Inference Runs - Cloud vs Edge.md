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

# Where Inference Runs - Cloud vs Edge

Parent: [[AI Inferencing]]

## 🪝 The Hook
The exact same AI feature — say, "summarize this document" — might run in a Google data center, on your laptop, on your phone, or on a tiny chip inside your car. Each place has completely different trade-offs on speed, cost, privacy, offline capability, and model quality. There isn't one right answer — smart AI products increasingly pick *different* places for different features, sometimes in the same app.

## ⚙️ Core Mechanism

Four rough places inference can happen:

### 1. Cloud (data center)

The default. You send your prompt to a provider's data center full of [[Inference Hardware]], they run the model, they send the reply back.

- ✅ Access to the largest models (only cloud has enough memory)
- ✅ Full [[Batching and Throughput]] economics — cheap per token at scale
- ✅ Model updates instantly for all users
- ❌ Your data leaves your device
- ❌ Requires internet
- ❌ Latency floor from network round-trip (~50–200ms)
- ❌ Per-request cost is real and recurring

Use when: model must be big, users are online, privacy isn't strict.

### 2. On-device (laptop / desktop)

Model lives on the user's machine. Runs on their CPU, GPU, or NPU. Examples: Apple Intelligence, Copilot+ PC features, local LLM tools like Ollama.

- ✅ Zero per-request cost (user's hardware)
- ✅ Data never leaves the machine (privacy)
- ✅ Works offline
- ✅ No network latency
- ❌ Model must fit in ~10–40GB (so mostly small models, heavily quantized)
- ❌ Slower per query — one user's hardware, no batching amortization
- ❌ Battery / heat cost for user
- ❌ Model updates require distributing gigabytes

Use when: privacy matters, offline matters, or task is small enough that a small model suffices.

### 3. On-device (phone)

Same idea but *much* tighter — model needs to fit in maybe 2–4GB, run at low power (battery), and share memory with all the other apps. Runs on the phone's NPU (Neural Engine on iPhone, Hexagon on Snapdragon).

- ✅ Everything on-device says, plus mobility
- ❌ Model has to be tiny (typically 1–8B params, INT4 or INT8 quantized)
- ❌ Even hotter thermal / battery limits
- ❌ Fragmented hardware (every Android chip is different)

Use when: photo enhancement, voice, keyboard suggestions, always-on background AI, everything the user does every second.

### 4. Edge devices (cameras, cars, IoT, robots)

Purpose-built silicon inside a device that never talks to a phone or cloud. Ring doorbell, autonomous car perception, factory-floor QA cameras, drones.

- ✅ Real-time, deterministic latency (no network!)
- ✅ Cheap per unit at scale (dedicated cheap chip)
- ❌ Model is often single-purpose, retrained per hardware
- ❌ Almost no flexibility (can't change model after shipping)

Use when: safety-critical (car braking) or connectivity-unreliable (drone), where cloud round-trip is unacceptable.

### The hybrid pattern (increasingly the norm)

Real products often use **multiple** locations at once:

- **Local first, cloud fallback** — small model handles simple queries fast on-device, kicks up to cloud for hard ones. (e.g., iPhone dictation → Siri cloud handoff.)
- **Router / cascade** — a tiny classifier decides which tier: local small model, mid cloud model, or top-tier expensive model. Saves cost on easy stuff.
- **Cloud for prefill, local for decode** — experimental. Send long context to cloud once, decode continuation locally after receiving the KV cache.
- **On-device draft + cloud verify** — [[Speculative Decoding]] where the draft model is on-device and the verify happens in cloud. Reduces cloud tokens.

### The privacy / regulation dimension

For healthcare, legal, financial, government AI — sending user data to a shared cloud can be flat-out illegal (HIPAA, GDPR, national security). This is one of the strongest forces pushing capable models on-device. The "small models are getting shockingly good" trend (Phi, Llama, Mistral small, etc.) largely exists *because* on-device is a huge market.

### The quality gap is closing

Historically, "on-device model" meant "much worse." That's changing fast:

- 2023: On-device model roughly equal to GPT-3.5. Cloud was leaps ahead.
- 2025: A 7B quantized model on a modern phone is roughly equal to GPT-4-class on many tasks.

The gap between cloud and on-device is narrowing every year. At some point the cloud advantage becomes only "we have the very latest frontier model" rather than "we have the only usable one."

## 🔗 Connects To
- [[AI Inferencing]]
- [[Inference Hardware]] — different chips power different tiers
- [[Quantization]] — the *reason* on-device is possible
- [[Cost of Inference]] — cloud has per-request cost, on-device has zero
- [[Latency - TTFT and TPOT]] — network is a hidden component of TTFT

## 💡 So What
When designing (or evaluating) any AI feature, ask: does this *need* to run in the cloud? Because "on-device" gives you privacy, offline, near-zero cost, and often lower latency — for free, if the task is small enough. The cloud should be a deliberate choice for tasks that *genuinely* need frontier-scale models.

## ❓ Open Question
Where's the actual crossover point at which "a good enough small model on device" beats "GPT-5 in the cloud" for most everyday tasks? Some argue it's already happened for the median query. If they're right, that reshapes the whole AI industry, from monetization models to hardware demand.

## 📚 Source
- Apple's Apple Intelligence technical writeups
- llama.cpp / MLX / ONNX Runtime documentation on on-device inference
- Qualcomm / Google whitepapers on mobile NPU performance

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
