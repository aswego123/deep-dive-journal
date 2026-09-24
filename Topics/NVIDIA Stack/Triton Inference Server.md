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

# Triton Inference Server

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
A compiled TensorRT-LLM engine is just a *file*. It doesn't listen on a port, doesn't queue requests, doesn't know how to batch ten users together, doesn't expose metrics. **Triton Inference Server** is the restaurant's front-of-house — it takes the compiled recipe (the engine) and turns it into an actual, running, request-handling service.

(Not to be confused with **Triton the language** — a Python-like GPU kernel compiler mentioned in [[How a Matmul Runs on a GPU]]. Same name, different NVIDIA project. Genuinely confusing; worth remembering.)

## ⚙️ Core Mechanism

### What Triton Inference Server does

Triton sits between the network and the compiled model. Its job:

1. **Accept requests** over HTTP/REST or gRPC
2. **Queue and batch them** — dynamic batching, in-flight batching for LLMs
3. **Route to the right backend** — the actual engine that executes the model
4. **Manage multiple models** — load, unload, version, run several models on one server
5. **Report health and metrics** — `/health`, Prometheus-format `/metrics`
6. **Scale across GPUs** — one Triton instance can spread models across several GPUs on a node

Kitchen version: the compiled engine is the recipe card. Triton is the entire front-of-house system — the host seating guests, the waiters taking orders, the ticket rail batching orders for the kitchen, the manager tracking how busy every station is.

### Backends — Triton doesn't care what's running

Triton's key design idea: it's **backend-agnostic**. A "backend" is a plugin that knows how to execute one *kind* of model:

- **TensorRT-LLM backend** — runs compiled engines from [[TensorRT and TensorRT-LLM]]
- **PyTorch (LibTorch) backend** — runs raw PyTorch models, no compilation needed
- **ONNX Runtime backend** — runs ONNX-format models
- **Python backend** — for custom pre/post-processing logic (tokenization, image resizing) written in plain Python
- **vLLM backend** — Triton can even wrap a completely different inference engine

This means one Triton server can serve a TensorRT-LLM-compiled chat model, a PyTorch vision model, and a Python-based reranker — all from the same process, same port, same monitoring.

### Dynamic batching — the general version

Recall [[Batching and Throughput]]: batching many requests together dramatically improves GPU utilization. Triton's **dynamic batcher** does this automatically for any backend:

- Requests arrive at slightly different times
- Triton waits a small configurable window (e.g., a few milliseconds)
- Groups whatever arrived into one batch, sends it to the backend together
- For LLM-specific in-flight/continuous batching, the TensorRT-LLM backend handles the more complex per-token scheduling itself (see [[Prefill and Decode]])

Trade-off knob: `max_queue_delay` — wait longer, get bigger batches (better throughput, worse latency). Same trade-off as always.

### Model repository — how Triton finds models

Triton expects models laid out in a specific folder structure:

```
model_repository/
├── llama3-70b/
│   ├── config.pbtxt          ← how to batch, GPU assignment, inputs/outputs
│   ├── 1/                    ← version 1
│   │   └── model.plan        ← the compiled TensorRT-LLM engine
│   └── 2/                    ← version 2 (hot-swappable)
│       └── model.plan
└── reranker/
    ├── config.pbtxt
    └── 1/
        └── model.onnx
```

`config.pbtxt` declares input/output tensor shapes, batching policy, which GPU(s) to use, instance count (how many copies to run in parallel). Triton watches this directory and can **hot-reload** new model versions without downtime — old version keeps serving until the new one is ready.

### Ensembles and pipelines

Real inference often needs multiple steps: tokenize → embed → generate → detokenize → safety-check. Triton supports **ensemble models** — a declared pipeline of multiple models/steps, executed as one logical request, with Triton handling the data hand-off between stages internally. Useful for RAG pipelines, multi-model agents, or anything with pre/post-processing around the core LLM.

### Metrics and observability

Triton exposes a Prometheus-compatible `/metrics` endpoint out of the box: queue time, inference time, GPU utilization, batch size distribution, request counts per model/version. This is what feeds the autoscaling decisions covered in [[NIM Operator on Kubernetes]] — you can't autoscale on "tokens/sec" if nothing is measuring it.

### Where Triton sits in the stack

```
Client request (HTTP/gRPC)
        ↓
Triton Inference Server
  ├── Scheduler / dynamic batcher
  ├── Model repository manager
  └── Backend (TensorRT-LLM, PyTorch, ONNX, Python, ...)
        ↓
   Compiled engine (see TensorRT and TensorRT-LLM)
        ↓
   CUDA + cuBLAS/cuDNN/NCCL + GPU hardware
```

### Why NIM wraps Triton instead of replacing it

[[NIM - NVIDIA Inference Microservices|NIM]] containers use Triton internally (in most cases) as the serving layer, with a thin OpenAI-compatible API shim on top and automatic engine selection baked in. NIM doesn't reinvent request queuing, batching, or model management — it packages Triton (already a solved, battle-tested problem) plus the right pre-built engine plus a friendlier API.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[TensorRT and TensorRT-LLM]] — the compiled engine Triton usually serves
- [[Batching and Throughput]] — Triton's dynamic/in-flight batching implements this directly
- [[Prefill and Decode]] — Triton's scheduler has to juggle both phases across many requests
- [[NIM - NVIDIA Inference Microservices]] — the layer built on top of Triton
- [[Cost of Inference]] — batching configuration here directly sets throughput and cost

## 💡 So What
"Model serving" and "model compilation" are two different jobs done by two different tools — TensorRT-LLM compiles, Triton serves. Confusing the two is the #1 way people misdiagnose a slow inference deployment: sometimes the engine is fine and the *batching config* is the problem, or vice versa.

## ❓ Open Question
Triton is extremely general-purpose (any backend, any model type). Purpose-built LLM servers like vLLM's own server are simpler and sometimes faster for the *specific* LLM-serving case. Is Triton's generality a long-term advantage (one server for everything) or a disadvantage (extra complexity for a job that's increasingly just "serve an LLM")?

## 📚 Source
- NVIDIA Triton Inference Server documentation and GitHub repository
- Triton "Architecture" docs — model repository, backends, ensembles
- NVIDIA developer blog posts on dynamic batching configuration

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
