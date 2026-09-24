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

# cuBLAS cuDNN and NCCL

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
Nobody at OpenAI or Meta writes their own matmul kernel from scratch. They call a library that NVIDIA spent a decade hand-tuning. Three libraries in particular quietly power almost every AI model you've ever used: **cuBLAS** (matrix math), **cuDNN** (neural network ops), and **NCCL** (talking between GPUs). This chapter is about the "ready-made spice rack" every kitchen uses instead of grinding their own spices.

## ⚙️ Core Mechanism

### cuBLAS — the matrix math library

**BLAS** = Basic Linear Algebra Subprograms, a decades-old standard interface for matrix/vector math (predates GPUs entirely). **cuBLAS** is NVIDIA's GPU-accelerated implementation.

What it gives you:
- `GEMM` (GEneral Matrix Multiply) — the single most important AI operation, `C = αAB + βC`
- Batched GEMM — many independent matmuls in one call (essential for [[Batching and Throughput]])
- Support for FP32, FP16, BF16, FP8, INT8 — automatically routes to Tensor Cores when the shapes/precision qualify

Under the hood, cuBLAS has literally thousands of hand-tuned kernel variants for different matrix shapes, picked via **heuristics or autotuning** at runtime. This is the same tiling strategy from [[How a Matmul Runs on a GPU]], just pre-built so you don't write it yourself.

**cuBLASLt** is the newer, more flexible version — lets you fuse extra operations (bias add, activation function) into the matmul call itself, avoiding a separate pass over memory. Fusion = fewer trips to HBM = faster, per [[GPU Memory Hierarchy - HBM L2 SRAM Registers]].

### cuDNN — the neural network library

**cuDNN** (CUDA Deep Neural Network library) covers operations that aren't plain matmul but show up constantly in neural nets:

- Convolutions (for CNNs, vision models)
- Normalization (BatchNorm, LayerNorm, RMSNorm)
- Activation functions (ReLU, GELU, SiLU)
- Pooling
- **Fused attention kernels** — modern cuDNN ships optimized flash-attention-style kernels

Why this matters for LLMs specifically: **attention** is not a plain matmul — it's matmul + softmax + masking + another matmul, all needing to happen without materializing giant intermediate tensors in HBM. cuDNN's fused attention kernels (and FlashAttention, which inspired them) do this whole sequence while keeping data in shared memory/registers as long as possible. Framework code (PyTorch, JAX) calls straight into these.

### NCCL — talking between GPUs

**NCCL** (NVIDIA Collective Communications Library, pronounced "nickel") implements the **collective operations** mentioned in [[NVLink NVSwitch and InfiniBand]] — AllReduce, AllGather, ReduceScatter, Broadcast, P2P send/recv — tuned for NVIDIA's specific interconnect topology.

Why you can't just "send bytes" naively:
- An 8-GPU AllReduce done naively (each GPU sends to a central node) creates a bottleneck at that one node
- NCCL implements **ring** and **tree** algorithms that spread communication evenly across all available NVLink/InfiniBand paths
- NCCL is **topology-aware** — it detects at startup whether GPUs are on the same NVSwitch, different nodes, etc., and picks the best algorithm automatically

**Ring-AllReduce** (the classic algorithm): arrange N GPUs in a logical ring. Each GPU sends a chunk to its neighbor while receiving from the other neighbor, `2×(N-1)` steps total, and every GPU ends up with the full sum using the theoretically minimal amount of data transferred per link. This is the algorithmic backbone of both training (gradient sync) and multi-GPU inference (tensor-parallel activation sync).

**Where NCCL shows up in inference specifically:**
- Tensor parallelism ([[Parallelism]]) splits one layer's weights across GPUs — after each split matmul, an **AllReduce** stitches the partial results back together. This happens dozens of times per forward pass. NCCL performance directly determines multi-GPU inference speed.
- This is why [[NVLink NVSwitch and InfiniBand]] bandwidth matters so much for serving big models — every layer, every token, NCCL is doing a synchronization that has to complete before the next layer can start.

### How they stack together

```
Your model code (PyTorch)
        ↓
cuDNN (attention, norm, activation)   cuBLAS (matmul)   NCCL (multi-GPU sync)
        ↓                                  ↓                    ↓
                    CUDA runtime + driver
        ↓
              GPU hardware (SMs, Tensor Cores, NVLink)
```

None of these libraries know anything about "language models" or "chat." They only know matrices, tensors, and communication patterns. Every clever thing an LLM does is built from these primitives, called millions of times per second.

### Why this matters even if you never write CUDA

Framework benchmarks that look identical on paper (same model, same GPU) can differ 2× in real throughput purely based on **which version of cuBLAS/cuDNN/NCCL** is installed, and whether the framework is calling the fused/fast code paths or falling back to generic ones. "Update your CUDA/cuDNN version" is genuinely one of the highest-value performance fixes in ML engineering.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[CUDA and the Driver]] — the runtime these libraries sit on top of
- [[How a Matmul Runs on a GPU]] — cuBLAS is this chapter's ideas, pre-built
- [[NVLink NVSwitch and InfiniBand]] — the hardware NCCL rides on
- [[Parallelism]] — NCCL AllReduce is the concrete mechanism behind tensor parallelism
- [[TensorRT and TensorRT-LLM]] — builds on top of these libraries, adds compilation and fusion
- [[Prefill and Decode]] — cuDNN's fused attention kernels matter enormously for both phases

## 💡 So What
When a multi-GPU inference setup feels "slower than expected," the first two things worth checking are: is NCCL using the fastest available path (NVLink vs falling back to something slower), and is the framework calling fused cuDNN/cuBLASLt kernels or a naive fallback. These library-level details often explain gaps that "better hardware" wouldn't fix.

## ❓ Open Question
NCCL is proprietary and NVIDIA-specific. AMD has RCCL (a NCCL-compatible reimplementation) and there are vendor-neutral efforts (UCC). How mature is cross-vendor collective communication today — could a mixed AMD/NVIDIA cluster ever really work well?

## 📚 Source
- NVIDIA cuBLAS, cuDNN, and NCCL official documentation
- "Bringing HPC Techniques to Deep Learning" — Andrew Gibiansky (2017) — ring-allreduce explained clearly
- NCCL GitHub repository and its architecture docs

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
