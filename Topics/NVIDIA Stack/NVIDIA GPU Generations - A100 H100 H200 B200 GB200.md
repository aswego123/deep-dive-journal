---
date: 2026-09-23
topic: NVIDIA Stack
season: AI and Compute
tags:
  - deep-dive
  - ai
  - nvidia
  - gpu
status: seedling
---

# NVIDIA GPU Generations - A100 H100 H200 B200 GB200

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
Every 18 months NVIDIA drops a new chip, prices go up, benchmarks 2× or more, and everyone panics-buys. But *what actually changes* between A100 → H100 → H200 → B200 → GB200? Once you know the four dials that matter (memory, bandwidth, precision, interconnect), each generation's story becomes obvious in one sentence.

## ⚙️ Core Mechanism

### The four dials that actually matter

For inference, only four numbers really move the needle:

1. **HBM capacity** — bigger models fit; bigger batches fit
2. **HBM bandwidth** — decode goes faster (see [[How to Calculate GPU Inference - FLOPs Memory and Time]])
3. **New low-precision Tensor Core formats** — FP8, FP4 — more FLOPs per cycle
4. **NVLink bandwidth** — bigger "virtual GPU" domains, more GPUs act as one

Every generation moves some subset of these. Now the story.

### The generation table

| Chip | Year | Arch | Process | SMs | HBM | HBM BW | FP16 TFLOPs | New precision | NVLink BW |
|---|---|---|---|---|---|---|---|---|---|
| **A100** | 2020 | Ampere | 7nm | 108 | 40 or 80 GB HBM2e | ~2.0 TB/s | ~312 | TF32, BF16 | 600 GB/s (3rd-gen) |
| **H100** | 2022 | Hopper | 4nm | 132 | 80 GB HBM3 | ~3.35 TB/s | ~989 | **FP8** (Transformer Engine) | 900 GB/s (4th-gen) |
| **H200** | 2024 | Hopper | 4nm | 132 | **141 GB HBM3e** | **~4.8 TB/s** | ~989 | (same as H100) | 900 GB/s |
| **B200** | 2024 | Blackwell | 4nm (dual die) | ~2× H100 | 192 GB HBM3e | ~8 TB/s | ~2,250 | **FP4** (2nd-gen Transformer Engine) | 1.8 TB/s (5th-gen) |
| **GB200** | 2024 | Blackwell + Grace | — | 2× B200 + 1 Grace CPU | 384 GB HBM3e | ~16 TB/s (pair) | ~4,500 | FP4 | 1.8 TB/s per B200 |

(Numbers are peak dense; sparse and FP8/FP4 numbers are higher.)

### Ampere (A100) — 2020

The first "big AI GPU." Introduced:
- **TF32** — a 19-bit format that trains at ~FP32 quality but on Tensor Cores
- Structured 2:4 **sparsity** support (throw away half the weights, ~2× speedup)
- Multi-Instance GPU (MIG) — carve one A100 into up to 7 isolated mini-GPUs
- HBM2e — big for the time

Kitchen version: NVIDIA's first purpose-built "restaurant chef." Still the workhorse of many enterprises through 2024–25 because they bought a lot of them.

### Hopper (H100) — 2022

The chip that made ChatGPT-scale training economically possible. Big changes:
- **Transformer Engine** — hardware-managed FP8 with per-tile scaling factors. Doubles throughput vs FP16 with minimal quality loss.
- **HBM3** — jumped to 3.35 TB/s
- **TMA (Tensor Memory Accelerator)** — async bulk copy engine, keeps Tensor Cores fed
- **DPX** — dynamic programming instructions (niche but real speedups)
- **4th-gen NVLink** — 900 GB/s per GPU; NVLink Switch enables larger domains
- **Confidential computing** — encrypted GPU memory

Kitchen version: same chef, but she got a copilot who preps ingredients before she asks (TMA), and learned to cook in fewer pans without losing quality (FP8).

### Hopper refresh (H200) — 2024

Same silicon as H100. One big change: **HBM3e instead of HBM3**.

- 141 GB instead of 80 GB
- ~4.8 TB/s instead of ~3.35 TB/s

Since inference is memory-bandwidth-bound (see [[Prefill and Decode]] and [[How to Calculate GPU Inference - FLOPs Memory and Time]]), the H200 gets roughly **1.4× the tokens/sec** of an H100 on the same model, with the same peak FLOPs. This is the clearest example of "compute isn't the bottleneck, memory is."

Also: bigger HBM means bigger models fit on one GPU, and bigger [[KV Cache]] means more concurrent users per GPU.

### Blackwell (B200) — 2024

A whole new architecture. And a whole new *packaging* trick:

- **Two dies connected by a 10 TB/s chip-to-chip link, presented to software as one GPU.** First time NVIDIA went multi-die on their AI flagship.
- **FP4 Tensor Cores** — second-gen Transformer Engine adds 4-bit precision with microscaling. On paper, ~5× the throughput of FP8 (though usable quality depends on the model).
- **HBM3e** — 192 GB, ~8 TB/s
- **5th-gen NVLink** — 1.8 TB/s per GPU
- **Reliability engine** — hardware that catches SRAM bit-flips (matters at scale)

Kitchen version: the whole kitchen just got twice as big and got a *thinner* set of knives (FP4) that can chop way more per hour if the recipe tolerates it.

### GB200 — the Grace-Blackwell super-chip

Not really "one GPU" — it's a **package containing 2× B200 GPUs + 1 Grace CPU (72 Arm cores)** all wired together with NVLink-C2C at 900 GB/s (CPU ↔ GPU) and 1.8 TB/s (GPU ↔ GPU).

Why bother with the CPU on-package?
- CPU can preprocess / route requests without a slow PCIe hop
- CPU can hold cold KV cache pages that don't fit in HBM, with fast CPU↔GPU transfer
- Simplifies system design — buy one board, not "server + GPUs"

GB200 is the building block of the **NVL72 rack** — 72 B200 GPUs (36 GB200 packages) wired as one giant NVLink domain, presented to the software as if they were one system. That's a discussion for [[NVLink NVSwitch and InfiniBand]] and [[DGX HGX and Superpods]].

### The "inference GPU" family (L4, L40S, etc.)

Not everything is an H100. NVIDIA also makes cheaper inference-focused GPUs:

- **T4** (Turing, 2019) — the workhorse for small-model inference for years. Cheap, low-power, INT8 tensor cores.
- **A10 / A10G** (Ampere) — mid-range inference, popular in cloud.
- **L4** (Ada Lovelace, 2023) — successor to T4. FP8 tensor cores, low power, media/video engines. Great for small LLMs and multimodal.
- **L40S** (Ada Lovelace, 2023) — the "big" Ada card. 48 GB, decent bandwidth, popular for mid-sized models and where H100s are unaffordable.

These matter because the entire NIM catalog (see [[NIM - NVIDIA Inference Microservices]]) ships pre-built engines for each of these targets. When a NIM starts up, it detects the chip and loads the right engine.

### Which chip for which job (rough guide)

| Chip | Best for |
|---|---|
| T4 / L4 | Small models, image/audio inference, cheap cloud instances |
| A10 / L40S | 7B–13B models, RAG, moderate throughput |
| A100 | Legacy fleets; 70B models with 2× cards; still fine |
| H100 | Frontier training + serious inference |
| H200 | Same as H100 but for inference-heavy shops (better $/tok) |
| B200 / GB200 | Frontier training; largest models; hyperscalers |

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Inside a GPU - SMs CUDA Cores and Tensor Cores]] — Tensor Core evolution IS the generation story
- [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] — HBM3 → HBM3e is the H100→H200 story
- [[NVLink NVSwitch and InfiniBand]] — GB200 NVL72 is meaningless without understanding this
- [[How to Calculate GPU Inference - FLOPs Memory and Time]] — the "why H200 > H100 for inference" math
- [[Quantization]] — the AI-side reason FP8/FP4 exist
- [[Inference Hardware]] — vendor-agnostic version

## 💡 So What
Don't just chase the newest chip. Ask what your workload needs: training frontier models → Blackwell. Serving mid-size LLMs cheap → L40S or H200. Legacy fleet → keep your A100s, they're still fine. The right chip is workload-shaped, not marketing-shaped.

## ❓ Open Question
Blackwell's dual-die architecture works because the on-package interconnect is *fast enough* to hide the two dies. But the same idea could keep growing — 4-die, 8-die packages, wafer-scale. Where does packaging hit a physical wall (thermals, yield, cost)? Cerebras claims wafer-scale works; NVIDIA is more conservative. Who's right long-term?

## 📚 Source
- NVIDIA A100 Tensor Core GPU Architecture whitepaper (2020)
- NVIDIA H100 Tensor Core GPU Architecture whitepaper (2022)
- NVIDIA Blackwell Architecture Technical Brief (2024)
- NVIDIA GB200 NVL72 product briefs

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
