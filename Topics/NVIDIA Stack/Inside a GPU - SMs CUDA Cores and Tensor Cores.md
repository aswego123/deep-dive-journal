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

# Inside a GPU - SMs CUDA Cores and Tensor Cores

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
"NVIDIA H100 has 16,896 CUDA cores." That's a marketing number. It doesn't tell you anything about how the chip *actually* works. The real story is that those cores live in groups called SMs, and inside each SM there are two very different kinds of workers: the **CUDA cores** (jack-of-all-trades) and the **Tensor Cores** (freakishly good at one thing). Understanding this split is understanding modern AI hardware.

## ⚙️ Core Mechanism

### The zoom-out view
A GPU is a hierarchy:

```
GPU package
 └── ~100+ Streaming Multiprocessors (SMs)
      └── Each SM has:
           ├── Warp schedulers (traffic cops)
           ├── ~128 CUDA cores (general FP32/INT math)
           ├── 4 Tensor Cores (specialized matmul units)
           ├── Shared memory / L1 cache
           ├── Register file
           └── Load/store units
```

Think of the whole GPU as a warehouse full of identical **workstations** (SMs). Each workstation has a bunch of general workers (CUDA cores) and a couple of specialist machines (Tensor Cores) sitting in the corner doing one thing brilliantly.

### Streaming Multiprocessor (SM) — the "workstation"

The SM is the fundamental unit of a GPU. An H100 has **132 SMs**. A consumer RTX 4090 has 128. A tiny embedded Jetson might have 8.

Each SM independently:
- Fetches instructions
- Manages a pool of threads (up to ~2048 at once on H100)
- Executes math on its CUDA cores + Tensor Cores
- Reads and writes memory
- Has its own private shared memory / L1 cache

If SMs were chefs at stations, the GPU is a kitchen with 132 stations, each capable of cooking independently but able to hand food between them via the pantry (global memory / HBM).

### Threads, warps, and blocks — the org chart

- **Thread** — one line cook doing one task
- **Warp** — a squad of exactly **32 threads** that all execute the same instruction at the same time (SIMT). This is the atomic unit of scheduling.
- **Thread block** — a group of warps (say, 4–32 of them) that live on the *same SM* and can share memory and synchronize
- **Grid** — the full army of thread blocks for one kernel launch, sprayed across all SMs

Programmers write a **kernel** (a small function) and say "launch this with a grid of 1024 blocks, each block with 256 threads." The GPU chews the grid into warps and streams them through SMs.

### CUDA cores — the general workers

A CUDA core does one basic math op per cycle — usually FP32, FP16, or INT32. Things like:
- `c = a + b`
- `c = a * b`
- `c = a * b + c` (fused multiply-add, FMA)

They're the workhorses for anything that *isn't* a big matmul: activation functions (ReLU, GELU), normalization, elementwise operations, softmax, attention masking, index arithmetic.

Each SM on H100 has 128 CUDA cores × 132 SMs ≈ 16,896 total. That's where the marketing number comes from.

### Tensor Cores — the specialists

Introduced in Volta (V100, 2017). This is where modern AI actually lives.

A single Tensor Core doesn't just multiply two numbers. **In one clock cycle it multiplies two small matrices and adds a third** — something like a 16×16 matmul with an accumulate. Millions of times more work per cycle than a CUDA core, but only if the work is "please matmul these matrices."

Tensor Cores are picky:
- Inputs must be in matrix tiles of specific shapes (e.g. 16×16, 8×32, 32×8 — varies by generation)
- Inputs must be in specific low-precision formats (FP16, BF16, FP8, INT8, INT4, and now FP4 on Blackwell)
- Data must be pre-arranged in shared memory in the right layout

If you use them right, they deliver **~10–20× the throughput** of CUDA cores on matmul-shaped work. If you use them wrong or feed them non-matmul work, they sit idle.

### Per-generation Tensor Core evolution

| Gen | GPU | Supported precisions | Rough peak (dense) |
|---|---|---|---|
| Volta | V100 | FP16 | ~125 TFLOPs |
| Turing | T4 | FP16, INT8, INT4 | ~130 TFLOPs |
| Ampere | A100 | FP16, BF16, TF32, INT8 | ~312 TFLOPs FP16 |
| Hopper | H100 | + FP8 (Transformer Engine) | ~989 TFLOPs FP8 |
| Blackwell | B200 | + FP4 | ~10 PFLOPs FP4 (sparse) |

Each generation adds a new low-precision format because AI keeps proving that ~lower precision is fine, and lower precision = smaller data = more parallel matmul per cycle. See [[Quantization]] for why the model tolerates this.

### How work actually flows in one SM

1. **Load** — threads pull weights + activations from HBM into shared memory (slow, big).
2. **Load again** — threads pull tiles from shared memory into registers (fast, tiny).
3. **Compute** — Tensor Cores chew tiles → accumulate results.
4. **Store** — threads push results back to shared memory → back to HBM.

The whole trick to fast GPU code is **doing lots of step 3 for each pair of step 1s**. Otherwise you're just moving data, not multiplying it. This is why [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] is the next chapter — the memory story dominates the compute story.

### Warp schedulers — the traffic cops
Each SM has 4 warp schedulers. Each cycle, each scheduler picks *one warp that's ready* and issues its next instruction. If a warp is stalled (waiting on memory), the scheduler picks a different ready warp. This is called **latency hiding through massive parallelism** — the GPU keeps busy by always having *some* warp ready to run.

That's why you want many, many threads in flight even if you don't "need" them — they're insurance against memory stalls.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Why GPUs Beat CPUs at AI]] — the "why so many workers" question this chapter answers structurally
- [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] — how those workers get fed
- [[How a Matmul Runs on a GPU]] — Tensor Cores in action
- [[NVIDIA GPU Generations - A100 H100 H200 B200 GB200]] — how SM count + Tensor Core capability changed each gen
- [[The Forward Pass]] — the AI workload this hardware was shaped for

## 💡 So What
When comparing GPUs, the specs that actually matter are: how many SMs, what Tensor Core precisions they support, and (from next chapter) memory bandwidth. Raw "CUDA core count" is nearly meaningless if the model runs mostly on Tensor Cores — which it does.

## ❓ Open Question
Blackwell's FP4 Tensor Cores claim ~10× the throughput of Hopper's FP8. But FP4 has only 16 possible values. How does that not destroy model quality? The trick is fine-grained scaling factors ("microscaling") — but I still need to see careful evals on how well it actually holds up.

## 📚 Source
- NVIDIA H100 Tensor Core Architecture whitepaper (2022)
- NVIDIA Blackwell Architecture Technical Brief (2024)
- "Volta: A New Era in AI" — NVIDIA blog, 2017

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
