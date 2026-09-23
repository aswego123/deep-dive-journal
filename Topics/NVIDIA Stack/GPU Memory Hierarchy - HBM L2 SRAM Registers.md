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

# GPU Memory Hierarchy - HBM L2 SRAM Registers

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
An H100 can do ~2000 trillion FP16 operations per second. And it *cannot* keep its Tensor Cores fed with data at that rate — not even close. The bottleneck on almost every AI workload isn't math, it's the memory pipe. This one fact shapes the entire GPU design.

## ⚙️ Core Mechanism

### The pyramid
A GPU has four main memory tiers. Each level down is bigger but slower.

```
                Registers          ~256 KB per SM     ~fastest, per-thread
                    ▲              cycle latency
                    │
         Shared Memory / L1         ~256 KB per SM    ~a few cycles
                    ▲              (programmable scratchpad)
                    │
                L2 Cache            ~50 MB on-die      ~dozens of cycles
                    ▲              (shared by all SMs)
                    │
             HBM (Global Memory)    80–192 GB         ~hundreds of cycles
                                    3–8 TB/s          off-chip stacks
```

Kitchen version:
- **Registers** — the ingredients literally in the chef's hand
- **Shared memory / SRAM** — the prep counter, one arm's reach
- **L2 cache** — the fridge in the kitchen, few steps away
- **HBM** — the walk-in pantry, needs a whole trip

Every trip further out takes exponentially more time. The whole art of GPU programming is *staying up the pyramid* as long as possible.

### HBM — the walk-in pantry

**HBM = High Bandwidth Memory**. Special DRAM stacks physically bonded next to the GPU die on the same package. Not "on the card" — literally millimeters from the compute.

Why HBM instead of normal DDR memory? DDR is designed for CPUs — long thin bus, ~100 GB/s per channel. HBM is designed for GPUs — insanely wide bus (1024+ bits per stack), stacked vertically to shorten wires.

| GPU | HBM type | Capacity | Bandwidth |
|---|---|---|---|
| A100 | HBM2e | 40 or 80 GB | ~2.0 TB/s |
| H100 | HBM3 | 80 GB | ~3.35 TB/s |
| H200 | HBM3e | 141 GB | ~4.8 TB/s |
| B200 | HBM3e | 192 GB | ~8 TB/s |

**Two numbers matter**: capacity ("does the model fit?") and bandwidth ("how fast can weights get to the SMs?"). See [[NVIDIA GPU Generations - A100 H100 H200 B200 GB200]] for why H200 is basically "H100 with more HBM and it's a big deal."

### L2 cache — the fridge

Modern GPUs have a big shared **L2 cache** (Hopper: ~50 MB, Blackwell: much larger and split per die). Reads to HBM get cached here. When many SMs need the same weights (which happens a lot in matmul), the second SM hits L2 instead of HBM. That's a ~5–10× bandwidth win, silently, on top of the raw HBM number.

L2 is not programmable — the hardware decides what to cache. But CUDA does give programmers *hints* (`__ldg`, cache modifiers) about what they'd like to keep hot.

### Shared memory / SRAM — the prep counter

Each SM has ~200–256 KB of on-chip SRAM that the programmer *controls directly*. Call it **shared memory** (from CUDA's point of view) or **L1** (from hardware's) — same silicon, split configurably between the two roles.

This is where matmul tiles live during multiplication. The trick of a fast matmul kernel: cooperatively load a tile into shared memory *once*, then have hundreds of threads reuse it many times before evicting. See [[How a Matmul Runs on a GPU]].

FlashAttention's whole speedup? Rearrange the attention math so more of it fits in shared memory, avoiding round trips to HBM. Same math, way less memory traffic.

### Registers — the hands

Each thread has a small pile of registers. Each SM has a *register file* of ~256 KB, split across the ~2048 threads running on it (so ~128 bytes per thread). Register access is essentially free — one cycle.

But: if a thread needs more registers than available, the compiler "spills" them to local memory (which actually lives in HBM). Sudden 100× slowdown. High-performance CUDA code obsessively minimizes register pressure.

### The bandwidth wall — the whole reason this chapter exists

Here's the punchline. To keep an H100's Tensor Cores fully fed doing FP16 matmul:

- Peak math: 989 TFLOPs FP16
- Each FP16 op needs 2 bytes of input on average
- Required bandwidth ≈ 989e12 × 2 ≈ **~2000 TB/s**
- Actual HBM bandwidth: **~3.35 TB/s**

That's a **~600× gap**. The chip can chew data 600× faster than it can be fed from HBM.

Which means: **the only way to keep Tensor Cores busy is to reuse each byte 600× before letting it go.** That reuse happens up the pyramid — in L2, in shared memory, in registers. If your workload can't reuse data that many times, you're memory-bound, and your fancy Tensor Cores idle.

This is called **arithmetic intensity** — FLOPs per byte moved. Matmul has high arithmetic intensity (O(N) reuse per byte). Decode-phase LLM inference (see [[Prefill and Decode]]) has *terrible* arithmetic intensity because batch size 1 means each weight is used exactly once per token. Which is exactly why decode is memory-bound and prefill isn't.

We compute all this properly in [[How to Calculate GPU Inference - FLOPs Memory and Time]].

### PCIe and NVLink — not memory but adjacent

Once HBM fills up, you have to talk to *other GPUs*. That's:
- **PCIe** — motherboard bus, ~64 GB/s per direction on PCIe 5.0. Slow. Only used when NVLink isn't available.
- **NVLink** — direct GPU-to-GPU wires, ~900 GB/s on Hopper, ~1.8 TB/s on Blackwell. See [[NVLink NVSwitch and InfiniBand]].

So there's actually a fifth (and sixth) tier under HBM: "other GPU's HBM (via NVLink)" and "other node's HBM (via InfiniBand)." Every hop, another order of magnitude slower.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Inside a GPU - SMs CUDA Cores and Tensor Cores]] — the compute side; this chapter is the feeding side
- [[How a Matmul Runs on a GPU]] — the whole tiling strategy exists because of this hierarchy
- [[How to Calculate GPU Inference - FLOPs Memory and Time]] — the math version of "the bandwidth wall"
- [[NVLink NVSwitch and InfiniBand]] — memory beyond one GPU
- [[Prefill and Decode]] — why decode is bandwidth-bound
- [[KV Cache]] — why this cache eats HBM so aggressively

## 💡 So What
When comparing GPUs, memory *bandwidth* usually predicts inference performance better than raw FLOPs. Two chips with equal FLOPs but different HBM bandwidth will feel very different on real LLM workloads. This is why "H100 → H200" (same compute, more HBM3e bandwidth) is a meaningful upgrade for inference even though the peak TFLOPs are identical.

## ❓ Open Question
CXL memory (attached over PCIe) claims to extend "GPU memory" cheaply with pooled DRAM. Does the latency penalty kill it for inference, or is there a real use case (spilling cold KV cache pages, maybe)? Need to check where CXL adoption actually is by 2027.

## 📚 Source
- NVIDIA Hopper Architecture whitepaper — sections on memory subsystem
- "FlashAttention" (Dao et al., 2022) — a case study in exploiting shared memory
- SemiAnalysis writeups on HBM economics

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
