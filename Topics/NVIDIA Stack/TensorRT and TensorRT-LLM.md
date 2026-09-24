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

# TensorRT and TensorRT-LLM

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
Take the exact same model weights and run them two ways: naively in PyTorch, or compiled through **TensorRT-LLM**. Same GPU, same weights, same math answer — but the compiled version can run **2–4× faster**. Nothing about the model changed. What changed is *how efficiently the same math gets scheduled onto the chip*. That's what a compiler buys you.

## ⚙️ Core Mechanism

### TensorRT — the general compiler

**TensorRT** is NVIDIA's inference compiler for neural networks in general (vision, recommendation, speech — not just LLMs). You give it a trained model (from ONNX, PyTorch, TensorFlow), it analyzes the whole computation graph, and outputs an **engine** — a file containing a highly optimized, GPU-specific execution plan.

What the compiler does that a naive framework doesn't:

- **Kernel fusion** — combine multiple operations (e.g., matmul + bias-add + activation) into one GPU kernel, avoiding round trips to HBM between each step (see [[GPU Memory Hierarchy - HBM L2 SRAM Registers]])
- **Precision calibration** — automatically convert parts of the model to FP16/FP8/INT8 where safe, using [[Quantization]] techniques
- **Kernel auto-tuning** — for each operation, benchmark several kernel implementations *on your specific GPU* and pick the fastest
- **Layer and tensor elimination** — remove dead code, constant-fold anything computable ahead of time
- **Memory planning** — pre-allocate and reuse buffers instead of allocating fresh memory per request

The output is locked to a specific GPU architecture (and often a specific GPU model) — you compile *for* an H100, and that engine won't run optimally (or sometimes at all) on an A100.

### TensorRT-LLM — the LLM specialist

Generic TensorRT struggles with LLM-specific realities: variable-length sequences, the KV cache, autoregressive generation, huge parameter counts split across GPUs. **TensorRT-LLM** is a purpose-built layer on top, adding:

- **In-flight batching** (NVIDIA's name for continuous batching, see [[Batching and Throughput]]) — new requests join and finished ones leave a running batch without waiting for a batch boundary
- **Paged KV cache** — same idea as vLLM's PagedAttention (see [[KV Cache]]); manages cache memory in fixed-size pages instead of one contiguous blob per request, dramatically reducing fragmentation
- **Built-in quantization** — FP8, INT8, INT4 (AWQ, GPTQ-style) baked into the compilation step
- **Speculative decoding support** — draft models, Medusa heads, etc. (see [[Speculative Decoding]]) as first-class compiled features
- **Multi-GPU support** — automatically inserts the NCCL AllReduce calls needed for tensor parallelism (see [[cuBLAS cuDNN and NCCL]]) directly into the compiled engine
- **Custom attention kernels** — fused, memory-efficient attention tuned per GPU generation

### The compilation workflow

```
1. Start: HuggingFace checkpoint (PyTorch weights)
2. Convert: weights → TensorRT-LLM's internal format, choose precision (FP8, INT4, etc.)
3. Build: trtllm-build compiles a GPU-specific "engine" file
      - picks kernels for your exact GPU (H100 vs L40S vs A10 differ)
      - fuses ops, plans memory, sets max batch/sequence size
4. Deploy: engine gets loaded by Triton (see next chapter) or wrapped in a NIM
5. Serve: engine executes forward passes at compiled speed
```

Step 3 can take minutes to hours depending on model size — this cost is paid *once*, at build time, not per request. That's the whole trade: expensive one-time compile, cheap fast serving forever after.

### Why "engine per GPU" matters (and sets up the NIM chapter)

An engine built for H100 uses H100-specific Tensor Core instructions, tuned tile sizes for H100's SM count, H100's specific memory bandwidth assumptions. Move that same engine to an L40S and it either won't run, or runs suboptimally.

This is exactly why [[NIM - NVIDIA Inference Microservices]] ships **multiple pre-built engines per model** — one per supported GPU — and picks the right one at container startup. TensorRT-LLM is the compiler that *produced* those engines ahead of time.

### Worked example — what compilation buys you

Rough, representative numbers for a 7B model on one H100, batch size 1, FP16:

| Setup | Tokens/sec (approx) |
|---|---|
| Naive PyTorch (`eager` mode) | ~40 |
| PyTorch + `torch.compile` | ~70 |
| TensorRT-LLM (FP16, no batching tricks) | ~90 |
| TensorRT-LLM (FP8 + in-flight batching, batch 64) | ~800+ (aggregate throughput) |

Single-request latency improves modestly (compiler + fusion wins). **Aggregate throughput** explodes once batching and quantization compound with the compiled engine — this is the [[How to Calculate GPU Inference - FLOPs Memory and Time]] roofline story in action: compilation removes waste, batching + quantization move you along the roofline.

### What TensorRT-LLM is NOT

- Not a training framework — it only compiles for inference
- Not a serving/networking layer — it produces an engine; something else (Triton, or NIM's built-in server) has to expose it over HTTP/gRPC
- Not magic — a badly shaped model (unsupported ops, exotic architecture) may need custom plugin code to compile at all

### Open source and ecosystem

TensorRT-LLM is open-sourced on GitHub (Apache 2.0 license for the Python API, some components proprietary). Competing compilers exist — vLLM's own kernels, Hugging Face's TGI, SGLang — and the field is genuinely competitive; TensorRT-LLM isn't automatically the fastest for every model/GPU combination, but it usually leads on NVIDIA hardware specifically because NVIDIA has first-party access to chip internals.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[How a Matmul Runs on a GPU]] — kernel fusion is this idea applied at scale
- [[cuBLAS cuDNN and NCCL]] — the libraries TensorRT-LLM's engines call into
- [[Quantization]] — the compile-time step that bakes in lower precision
- [[KV Cache]] and [[Batching and Throughput]] — in-flight batching + paged KV cache are TensorRT-LLM's implementations of these ideas
- [[Speculative Decoding]] — supported as a compiled feature
- [[Triton Inference Server]] — where compiled engines actually get served
- [[NIM - NVIDIA Inference Microservices]] — ships pre-compiled engines per GPU

## 💡 So What
"We're using TensorRT-LLM" is a meaningful performance claim, not just a brand name — it implies kernel fusion, GPU-specific tuning, and modern batching/caching are all baked in. When evaluating a model server, ask whether it's using a compiled engine (TensorRT-LLM, vLLM's compiled kernels) or running the model "eagerly" — that answer alone predicts a lot of the throughput.

## ❓ Open Question
Compiling an engine per GPU generation is powerful but operationally heavy — you need a build pipeline, storage for many engine variants, and rebuilds on every model or driver update. Is there a real trend toward "compile once, run anywhere" for LLM inference, or will GPU-specific compilation always be the performance-leading approach?

## 📚 Source
- NVIDIA TensorRT-LLM GitHub repository and documentation
- NVIDIA TensorRT documentation (general compiler)
- NVIDIA developer blog: "TensorRT-LLM: A Comprehensive Guide" posts

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
