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

# CUDA and the Driver

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
A GPU is a slab of silicon. It doesn't know what "PyTorch" is, doesn't know what a "matrix" is, doesn't even know what "on" means without help. **CUDA** is the entire reason a programmer can type `model(x)` in Python and have that turn into billions of Tensor Core operations. It's the layer everything else in this book is built on top of.

## ⚙️ Core Mechanism

### What "CUDA" actually refers to (it's three things at once)

People say "CUDA" to mean three different layers, which causes constant confusion:

1. **CUDA the architecture** — the hardware design (SIMT, warps, SMs) covered in [[Inside a GPU - SMs CUDA Cores and Tensor Cores]]
2. **CUDA the driver** — a piece of software installed on your OS that talks to the physical GPU
3. **CUDA the toolkit / API** — the programming language extension (`__global__`, `<<<...>>>`), compiler (`nvcc`), and libraries developers actually write code against

This chapter is mostly about #2 and #3.

### The driver — the GPU's translator

The **NVIDIA driver** is a kernel-mode piece of software that:
- Talks directly to the GPU hardware (memory-mapped registers, command queues)
- Manages GPU memory allocation
- Schedules which process's work runs on the GPU when
- Exposes a stable API (the "CUDA Driver API") that everything above it uses

Every GPU has a **compute capability** number (e.g., Hopper = 9.0, Blackwell = 10.0) baked into the driver/toolkit relationship — it tells software "this chip supports these instructions." Code compiled for compute capability 9.0 might not run (or might not use new features) on older 7.x hardware.

Kitchen version: the driver is the head waiter who actually knows how to talk to this exact chef in this exact kitchen — knows their quirks, their tools, their pace. Without the head waiter, nobody outside the kitchen can place an order.

### The kernel — the unit of GPU work

A **CUDA kernel** is a function written to run on the GPU, launched from the CPU ("host") side:

```cuda
__global__ void addVectors(float* a, float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) c[i] = a[i] + b[i];
}

// Launch: 1024 blocks of 256 threads each = 262,144 threads
addVectors<<<1024, 256>>>(a, b, c, n);
```

That `<<<1024, 256>>>` is the launch configuration — how many thread blocks, how many threads per block. The driver takes this, schedules it across the GPU's SMs, and each thread computes `i` from its block/thread IDs to know which piece of data it owns. This is the raw mechanism behind [[How a Matmul Runs on a GPU]].

### Streams — doing more than one thing at once

A **CUDA stream** is a queue of GPU work that executes in order *within* the stream, but different streams can run **concurrently** — one stream's memory copy overlapping with another stream's compute.

Real inference servers use multiple streams so that:
- Copying the next batch's input to GPU memory (stream A)
- Overlaps with computing the current batch (stream B)
- Overlaps with copying the previous batch's output back to CPU (stream C)

This is exactly the kind of pipelining discussed in [[How a Matmul Runs on a GPU]], but at a whole-request level instead of within one kernel.

### CUDA Graphs — skip the overhead

Launching a kernel isn't free — there's a few microseconds of CPU-side overhead per launch to set up the call. For LLM inference, you might launch **hundreds of kernels per token** (one per layer, per operation). At high token rates, that launch overhead adds up.

**CUDA Graphs** let you record a whole sequence of kernel launches once, then **replay the entire graph with one call**. This slashes CPU-side overhead — critical for decode, where each step is small and overhead-sensitive (see [[Prefill and Decode]]). TensorRT-LLM uses CUDA Graphs heavily for exactly this reason.

### The toolkit and ecosystem

- **nvcc** — NVIDIA's CUDA compiler; compiles `.cu` files into GPU machine code (PTX, then SASS)
- **PTX** — an intermediate assembly-like language, portable across GPU generations; the driver JIT-compiles PTX into the exact chip's native instructions (SASS) at runtime if needed
- **cuda-toolkit** — the whole SDK: compiler, debugger (`cuda-gdb`), profiler (`Nsight Systems`, `Nsight Compute`), libraries

### Why almost nobody writes raw CUDA anymore

Writing a fast matmul kernel by hand takes real expertise. Most AI developers never touch CUDA directly — they use:

- **PyTorch / TensorFlow** — call into pre-written CUDA kernels (via cuDNN, cuBLAS) under the hood
- **Triton (the language)** — write Python-like code, compiler generates efficient CUDA/PTX automatically
- **TensorRT-LLM** — takes a whole model description and compiles an optimized engine, no CUDA code needed from the user

CUDA is the foundation. Almost everyone stands on libraries built on top of it. See [[cuBLAS cuDNN and NCCL]] for the next layer up.

### The moat

CUDA has existed since 2007. Fifteen-plus years of libraries, tooling, documentation, and — critically — **millions of engineers who already know it**. AMD's ROCm and Intel's oneAPI are real alternatives but chase a moving target with a much smaller ecosystem. This software depth, more than any single chip spec, is why NVIDIA has stayed dominant.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[Inside a GPU - SMs CUDA Cores and Tensor Cores]] — the hardware CUDA programs against
- [[How a Matmul Runs on a GPU]] — kernels in action
- [[cuBLAS cuDNN and NCCL]] — the libraries built on top of CUDA
- [[TensorRT and TensorRT-LLM]] — the compiler layer that hides CUDA from most users
- [[Prefill and Decode]] — CUDA Graphs matter most for decode's many small steps

## 💡 So What
When someone says "we need CUDA-compatible hardware," they don't just mean "an NVIDIA GPU" — they mean "a chip with 15+ years of driver, compiler, and library support behind it." That's a much bigger ask for a competitor than just matching FLOPs on a spec sheet.

## ❓ Open Question
PyTorch 2.0's `torch.compile` and Triton (the language) are trying to make "write fast GPU code" require zero raw CUDA knowledge. How close are they to matching hand-written CUDA/CUTLASS performance for the *specific* shapes LLM inference uses? Need to check current benchmarks.

## 📚 Source
- NVIDIA CUDA C++ Programming Guide (current)
- "CUDA by Example" — Sanders & Kandrot — still a solid intro
- NVIDIA CUDA Graphs documentation and blog posts

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
