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

# How a Matmul Runs on a GPU

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
Every AI benchmark is basically a matmul benchmark. So the single most important question in GPU performance is: *how does one big matrix multiplication turn into work spread across 132 SMs, each with 4 Tensor Cores, each with a bunch of threads, each with a handful of registers?* The answer is one word: **tiling**.

## ⚙️ Core Mechanism

### The problem
You want `C = A × B` where A is `[M×K]`, B is `[K×N]`, and C is `[M×N]`. For an LLM, M might be batch × sequence (~1000s), K is hidden dim (~4000), N is another hidden dim (~4000). That's tens of millions of dot products.

You have:
- 132 SMs, each capable of chewing on a slice
- Only ~256 KB of shared memory per SM
- HBM bandwidth that can't feed the Tensor Cores raw

The naive plan (compute each output cell as its own dot product, streaming from HBM) is a disaster: you'd reload A and B billions of times. Not memory-bound — memory-*destroyed*.

### The solution — three layers of tiling

Break the big matmul into a **nested hierarchy of tiles**, matching the memory hierarchy from [[GPU Memory Hierarchy - HBM L2 SRAM Registers]].

```
Big matmul (millions of ops)
  ↓ chop into
Block tiles       (e.g. 128×128)   → one per SM, lives in shared memory
  ↓ chop into
Warp tiles        (e.g. 64×64)     → one per warp, uses tensor cores
  ↓ chop into
Thread tiles      (e.g. 16×16)     → one per tensor core instruction (MMA)
```

Each level solves a different problem:

- **Block tile (SM level)** — one SM owns a 128×128 chunk of the output. It loads the corresponding stripes of A and B into shared memory *once*, then reuses them.
- **Warp tile (warp level)** — inside the block tile, a warp handles a 64×64 sub-chunk. Loads sub-tiles from shared memory into registers.
- **Thread tile / MMA** — the actual Tensor Core instruction. Each `wmma::mma_sync` (or in modern CUDA, `mma`) call does one small matmul (e.g. 16×16×16) using registers, in one cycle.

Kitchen version: the head chef gets an order for 10,000 pasta bowls (big matmul). She splits it into 100 orders of 100 bowls each and hands one to each station (SM). Each station splits its 100 bowls among 4 cooks (warps). Each cook uses a machine that plates 16 bowls at once (Tensor Core MMA).

### Why this works — the reuse math

Consider one SM doing a 128×128 output tile with K = 4096.

- **Data read from HBM**: 128×4096 (A stripe) + 4096×128 (B stripe) = ~1 MB
- **Math done**: 128×128×4096 = ~67 million multiply-adds
- **Arithmetic intensity**: ~67 FLOPs per byte

Now compare to naive (no shared memory): each of 128×128 output cells would independently read its own A row + B column. That's ~4 KB per cell × 16,384 cells = ~67 MB of memory traffic for the same 67 MFLOPs. Arithmetic intensity ~1.

The tile version does **~64× less memory traffic** for the same math. That's the difference between "memory-bound and slow" and "compute-bound and fast."

### The Tensor Core instruction — WMMA / MMA

At the bottom of the tiling hierarchy is one CUDA instruction that says:

> "Take these two small matrices in my registers. Take a third small accumulator matrix. Do `D = A × B + C`. Put the result back in registers."

Modern Hopper `wgmma` variants can operate on quite large tiles (64×256×16 or so). Blackwell variants are even larger and support new precisions (FP8, FP6, FP4). But the shape is always: **one asynchronous matmul-accumulate on a small tile using specialized silicon.**

Programmers rarely write these instructions by hand. They use:
- **CUTLASS** — NVIDIA's C++ template library for writing peak-performance matmul kernels
- **Triton** (the language, unrelated to Triton Inference Server) — a Python-like DSL that compiles to these instructions
- **cuBLAS** — a library of hand-tuned kernels for every common shape, see [[cuBLAS cuDNN and NCCL]]

### The pipeline dance

Modern kernels don't do "load → compute → store" serially. They *overlap* stages:

```
Stage:  Load tile K   |   Compute tile K   |   Store tile K
Time →  Load tile K+1 |   Compute tile K   |   Store tile K-1
        Load tile K+2 |   Compute tile K+1 |   Store tile K
```

While Tensor Cores compute tile K, async memory copies bring in tile K+1 from HBM to shared memory. Hopper introduced the **TMA (Tensor Memory Accelerator)** — a dedicated unit that does these bulk async copies without occupying the compute threads. Big deal for keeping the pipeline full.

### Batched and grouped matmul

LLM inference rarely has one giant matmul. It has *many* medium-sized matmuls (one per layer, per attention head, per batch element). Modern kernels **batch these together** into one launch — either as a plain batched GEMM or as a "grouped GEMM" where each item can have a different shape. This is critical for [[Batching and Throughput]].

### When it goes wrong

You can lose 90% of peak performance to:
- **Bad tile shapes** — using a 128×128 tile when your actual matrix is 130 wide leaves tiles half-empty
- **Register pressure** — asking for too many registers per thread; the compiler spills to memory, everything crawls
- **Bank conflicts** — shared memory has 32 banks; if threads all hit the same bank, they serialize
- **Warp divergence** — even a single `if` inside the kernel can halve throughput
- **Wrong precision** — using FP32 accumulators when FP16 would suffice, halving Tensor Core throughput
- **Bad launch config** — too few blocks means idle SMs

Which is why 95% of AI engineers never write matmul kernels — they use libraries written by people who've spent years staring at NCU (Nsight Compute) profiles.

### Modern kernels most LLMs actually use

- **cuBLAS / cuBLASLt** — general-purpose, autotuned
- **CUTLASS** — customizable, source available, basis for many hand-rolled kernels
- **cuDNN** — includes fused attention/convolution kernels
- **FlashAttention** (2/3) — attention as one big fused kernel, avoids materializing the giant N×N matrix
- **TensorRT-LLM's** kernels — see [[TensorRT and TensorRT-LLM]]; often the fastest for the specific model shapes it targets

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Inside a GPU - SMs CUDA Cores and Tensor Cores]] — the hardware this maps onto
- [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] — the entire reason for tiling
- [[How to Calculate GPU Inference - FLOPs Memory and Time]] — the numeric consequences
- [[cuBLAS cuDNN and NCCL]] — the library layer that ships these kernels
- [[TensorRT and TensorRT-LLM]] — the compiler that picks/generates the best kernel
- [[The Forward Pass]] — the AI workload driving all this

## 💡 So What
Every "AI made faster" headline is downstream of somebody finding a better tiling, fusion, or async-copy trick for these matmul kernels. When benchmarks show wildly different tokens/sec on the "same" hardware, kernel quality is often the difference.

## ❓ Open Question
Autotuning matmul kernels (search-based tuners like AITemplate, Triton autotuner) is now competitive with hand-tuned CUTLASS in many cases. Does hand-tuning eventually disappear, or do the very last 10% of perf always require a human?

## 📚 Source
- CUTLASS repository and its `docs/` — the clearest walkthrough of tile hierarchy that exists
- "FlashAttention-2" (Dao, 2023) and "FlashAttention-3" (Shah et al., 2024)
- NVIDIA Hopper Tensor Memory Accelerator whitepaper section

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
