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

# Inference Hardware

Parent: [[AI Inferencing]]

## 🪝 The Hook
Everyone says "AI runs on GPUs." But NVIDIA already sells four different classes of chips *just* for AI, and Google, Amazon, Apple, and half a dozen startups make their own. Meanwhile that same AI can also run on your CPU, or your phone's NPU, or a tiny chip in a Ring doorbell. Why so many? Because inference has a very specific shape of workload, and different chips are optimized for different parts of that shape.

## ⚙️ Core Mechanism

From [[Prefill and Decode]] we know inference is *two* different workloads:
- **Prefill** = massive parallel math on a lot of tokens at once → **compute-bound**
- **Decode** = tiny math per step but loading a huge model from memory every step → **memory-bandwidth-bound**

Every chip you can run AI on is a trade-off along these two axes: how much math per second (FLOPs), how much memory bandwidth (bytes per second), how much memory total (GB), and how much it costs per hour.

### The main hardware categories

#### 1. Data-center GPUs (NVIDIA H100/H200/B200, AMD MI300)

The workhorses of cloud AI. Massive math (thousands of TFLOPs), huge memory bandwidth (3+ TB/s on H100, 8+ TB/s on B200), lots of HBM memory (80–192 GB per chip), and they can be networked into pods of 8, 72, or more chips talking to each other over ultra-fast interconnect (NVLink).

Good at: everything, but especially high-throughput serving of big models with big batches.

Bad at: cost (\$30k+ each), power (~700W each), scarcity, and being wildly overkill for small models.

#### 2. Data-center accelerators (Google TPU, AWS Trainium/Inferentia, custom silicon)

Different companies bet on different chip designs. TPUs are Google's answer to GPUs — very good at big matmuls, especially networked. Inferentia (AWS) is *specifically* built for inference: less flexible than a GPU, but cheaper per token for the exact workloads it targets.

Good at: cost efficiency when workload matches the design. Vertical integration with a specific cloud.

Bad at: portability — code often has to be rewritten for each vendor's stack.

#### 3. Consumer GPUs (NVIDIA RTX 4090/5090, AMD RX cards)

Gamer cards that happen to be great at AI too. Way less memory (24GB on a 4090 vs 80GB on H100), less memory bandwidth, but ~1/10 the cost. Combined with [[Quantization]], they can run 7B–70B models locally.

Good at: hobbyist / small startup inference, dev, on-prem private AI.

Bad at: multi-user serving at scale (memory too small for many KV caches), not officially "supported" for data-center use.

#### 4. CPUs (Intel, AMD, Apple Silicon)

Modern CPUs have gotten scarily good at AI thanks to wide vector units (AVX-512, AMX) and, on Apple Silicon, unified memory that shares up to 128GB+ with the CPU. Not fast enough for cloud-scale serving, but plenty for local single-user use, especially with quantized small models.

Good at: on-device inference, embedded, private use, batch processing where latency doesn't matter.

Bad at: throughput vs GPU per dollar in the cloud.

#### 5. NPUs (Neural Processing Units)

Purpose-built AI chips inside phones (Apple Neural Engine, Qualcomm Hexagon), laptops (Intel/AMD NPUs), and small devices. Very low power, very small memory, only handle quantized models. Optimized for the specific patterns of on-device AI: image recognition, voice, small language models.

Good at: battery life, always-on features, privacy (data stays on device).

Bad at: anything requiring big models or big context — they can't fit.

#### 6. Edge / embedded (Coral TPU, Jetson, custom ASICs)

Tiny chips for cars, cameras, robots. Extreme power and cost constraints. Often handle only one model, only one task.

### The two numbers that actually matter

For inference, the two specs to look at first, before anything else:

1. **Memory capacity (GB)** — can the model even fit? A 70B model in FP16 needs ~140GB, in INT4 needs ~35GB. If your chip can't hold the model, nothing else matters.
2. **Memory bandwidth (GB/s)** — this sets your decode speed. A rough estimate: **max decode tokens/sec ≈ (memory bandwidth) / (model size in bytes)**. On an H100 with 3 TB/s reading a 70B INT4 model (~35 GB): 3000 / 35 ≈ 85 tok/s theoretical peak.

Raw math throughput (FLOPs) mostly matters for prefill and for very large batches. For everyday chat with small batches, memory bandwidth dominates.

### Networking is a hidden hardware layer

For models too big to fit in one chip, chips have to talk to each other during inference. NVLink (NVIDIA's chip-to-chip) is ~900 GB/s; InfiniBand or Ethernet between servers is ~100–400 GB/s. This matters a lot for models spread across GPUs — the interconnect can become the bottleneck. Same issue we saw in [[Parallelism]] but on the inference side.

## 🔗 Connects To
- [[AI Inferencing]]
- [[Parallelism]] — training's version of the "how many chips, how connected" problem
- [[Prefill and Decode]] — the reason memory bandwidth matters so much
- [[Quantization]] — makes smaller/cheaper hardware viable
- [[Where Inference Runs - Cloud vs Edge]] — the deployment side of the hardware question

## 💡 So What
When picking a model to deploy, the *hardware* it needs is often a bigger constraint than the model itself. "This 70B model is great" is meaningless without knowing "will it fit on the chip we can afford, at the memory bandwidth we need for our TPOT target?"

## ❓ Open Question
Are dedicated inference ASICs (Groq's LPU, Cerebras wafer-scale) actually a durable advantage, or does NVIDIA's software moat (CUDA) mean flexible GPUs win every time in the long run? Groq's speed numbers are staggering but hardware startups have a rough history.

## 📚 Source
- NVIDIA H100 whitepaper
- Google TPU v5e/v5p architecture posts
- Groq LPU technical writeups

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
