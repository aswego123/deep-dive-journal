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

# Why GPUs Beat CPUs at AI

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
A modern CPU costs about the same as a modern GPU. Yet on AI work, one GPU does the job of *hundreds* of CPUs. Why? The CPU is a genius. The GPU is a mob. Genius doesn't help when the job is "do a billion tiny multiplications, all independent, all at once."

## ⚙️ Core Mechanism

### The one-line answer
CPUs are optimized for **one hard task fast**. GPUs are optimized for **a million easy tasks in parallel**. AI is a million easy tasks in parallel. Match made in silicon heaven.

### Kitchen version
- **CPU chef** — 8 to 16 elite chefs. Each can invent a new recipe on the fly, taste, adjust, make judgment calls. Slow but brilliant.
- **GPU chef** — 10,000+ line cooks. Every cook does *exactly the same tiny thing* — chop this one carrot — but 10,000 carrots get chopped at the same instant.

Ask them to write a novel: CPU wins easily. Ask them to chop enough carrots for a stadium: the GPU is 500× faster and it isn't close.

### What "AI math" actually looks like
Almost every operation in a modern neural network is a **matrix multiply** (matmul). And matmul is *humiliatingly parallel*:

- Every output cell is an independent dot product
- No output cell needs to know what any other output cell is doing
- The same tiny operation (multiply, add) repeats billions of times

If your job is "do this same tiny thing a billion independent times," a mob of cooks crushes a team of geniuses. Full stop.

### The technical name — SIMT
NVIDIA calls their model **SIMT** — Single Instruction, Multiple Threads. One instruction ("multiply these two numbers") gets broadcast to 32 threads at once (a "warp"). All 32 execute the same instruction on different data. The GPU has *thousands* of warps in flight simultaneously.

CPUs also have parallelism (SIMD via AVX, multi-core, hyperthreading), but at nowhere near this scale. A top CPU might have 64 cores × 8-wide SIMD ≈ 512 parallel ops per cycle. An H100 has ~16,000+ CUDA cores plus tensor cores that do way more per cycle. Different universe.

### Why not just make CPUs wider?
Because CPUs spend most of their silicon on stuff that helps *one thread go fast*:

- Huge branch predictors ("what's the code probably about to do?")
- Deep out-of-order execution ("reshuffle instructions to hide waits")
- Big L1/L2/L3 caches ("keep this thread's data close")
- Complex instruction decoders

Every one of those transistors is a transistor *not* doing math. On a GPU, most of the die is math units and memory pipes. There's very little cleverness per thread. That's the whole trade.

### Where CPUs still win
- **Branchy code** — lots of "if this, else that." GPUs hate branches; whole warps stall when threads take different paths ("warp divergence").
- **Sequential logic** — anything where step 2 must wait for step 1. Can't parallelize what's inherently serial.
- **Small tasks with high overhead** — launching a GPU kernel costs microseconds. If your job is 5μs of math, the CPU wins before the GPU has finished stretching.

Which is why real inference systems (see [[Triton Inference Server]]) run *both* — CPU handles request routing, tokenization, scheduling; GPU handles the actual matmul storm.

### A rough performance comparison

| Job type | CPU | GPU (H100) | Winner |
|---|---|---|---|
| Compile a program | Fast | Terrible | CPU |
| Parse JSON | Fast | Awkward | CPU |
| Multiply two 4096×4096 matrices | ~seconds | ~milliseconds | GPU by 100–1000× |
| Run a forward pass on 70B model | Impossibly slow | ~30–100 tok/s | GPU by 1000× |
| Preprocess an image batch | OK | Blazing | GPU |

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Inside a GPU - SMs CUDA Cores and Tensor Cores]] — the "mob of cooks" in structural detail
- [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] — how the mob gets fed
- [[The Forward Pass]] — the AI-side of why matmul is *the* operation
- [[Inference Hardware]] — the vendor-agnostic version of this chapter

## 💡 So What
When someone asks "why can't we run this AI on a normal server?" the honest answer is: you can, if you don't mind waiting 500× longer. GPUs aren't magic — they're just the right shape of chip for the shape of the problem. Whenever a workload *doesn't* look like matmul-storm (agents doing many small model calls, weird branchy control flow), the GPU advantage shrinks.

## ❓ Open Question
NPUs (phone AI chips) go even further in the "throw out generality, keep only matmul" direction. Are they the endgame — GPUs eventually replaced by even more specialized silicon per task — or does CUDA's programmability keep GPUs on top?

## 📚 Source
- "GPU Computing" — John D. Owens et al., Proceedings of the IEEE (2008), still the cleanest explanation of SIMT
- NVIDIA CUDA Programming Guide, ch. "Programming Model"
- Companion to [[Inference Hardware]]

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
