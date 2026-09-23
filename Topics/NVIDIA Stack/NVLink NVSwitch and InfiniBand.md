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

# NVLink NVSwitch and InfiniBand

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
"We have 10,000 GPUs" is only impressive if those GPUs can actually talk to each other. A cluster where every chip is a genius but the wiring is a garden hose is worse than a smaller cluster with fire-hose wiring. Interconnect is often the *real* differentiator, and NVIDIA has spent 10 years building three layers of it: **NVLink** (chip-to-chip), **NVSwitch** (fabric), and **InfiniBand** (node-to-node).

## ⚙️ Core Mechanism

### Why interconnect exists at all

Even a huge GPU like B200 has "only" 192 GB. Big models don't fit. Even models that fit on one GPU serve better across many GPUs (bigger batches, sharded work). So the entire modern AI story assumes **many GPUs cooperating on one problem**.

Cooperation needs communication. Two questions determine everything:

- **How fast** can GPU A send bytes to GPU B?
- **How many hops** are there between them?

Three tiers of connection answer this at three physical distances.

### Kitchen version

- **NVLink** — chefs standing right next to each other passing plates directly, hand to hand
- **NVSwitch** — a giant lazy-Susan in the middle of a station letting any 8 chefs at that station pass plates to any other
- **InfiniBand** — a fast pneumatic tube system connecting one station's kitchen to another kitchen down the hall

Each tier is slower than the one above. Latency-sensitive stuff stays local; only reductions ("everyone tell me your total") need to cross tiers.

### Tier 1 — NVLink: chip-to-chip

**NVLink** is a set of high-speed serial links wired directly between GPU packages on a shared board. Not through PCIe, not through the CPU. Just wires from one GPU to the next.

Per generation:

| Gen | GPUs | Per-GPU BW (aggregate) | Notes |
|---|---|---|---|
| NVLink 3 | A100 | 600 GB/s | 12 links × 50 GB/s |
| NVLink 4 | H100 | 900 GB/s | 18 links × 50 GB/s |
| NVLink 5 | B200 | 1,800 GB/s | doubled per-link + more links |

To compare: PCIe 5.0 x16 = ~64 GB/s each way. NVLink is **~15–30× faster** than the motherboard bus GPUs normally use to talk. This is why NVLink-connected GPUs feel qualitatively like "one bigger GPU" while PCIe-connected GPUs feel like separate machines.

### Tier 2 — NVSwitch: the fabric

If you only had NVLinks between GPUs, you'd have to wire them in a mesh — 8 GPUs = 28 direct connections. Not scalable. Enter **NVSwitch**: a purpose-built chip that acts like a network switch just for NVLinks.

An H100 HGX baseboard has 4 NVSwitches. Every GPU has NVLink connections to every NVSwitch. Every switch is fully non-blocking. Result: **all 8 GPUs can talk to all other 7 GPUs simultaneously at full 900 GB/s each**. No congestion.

This 8-GPU pod acts to software like one 640 GB "virtual GPU" — collective operations (all-reduce, all-gather) happen at NVLink speed, not PCIe speed. Critical for tensor parallelism and NCCL (see [[cuBLAS cuDNN and NCCL]]).

### The big new thing: NVLink Switch (external)

For a long time, NVLink was stuck inside one server (max 8 GPUs). Blackwell changes this dramatically:

- **NVLink Switch (5th-gen)** is a **rack-level** switch
- **GB200 NVL72** wires **72 Blackwell GPUs across 18 servers** into one NVLink domain
- Every GPU-to-GPU pair in that rack talks at 1.8 TB/s
- Total NVLink bandwidth in one NVL72 rack ≈ **130 TB/s**

Software sees 72 GPUs as one giant machine with ~13 TB of unified HBM. This is a fundamentally different scale of "single GPU." Training and inference algorithms that assumed "8 GPUs max" now have to relearn.

### Tier 3 — InfiniBand: node-to-node

Once you leave the NVLink domain (whether 8 GPUs on HGX or 72 on NVL72), you're crossing a network. NVIDIA's answer: **InfiniBand**, a low-latency, high-bandwidth network fabric they own via the Mellanox acquisition (2020).

- **ConnectX-7 NIC**: 400 Gb/s (~50 GB/s) per port
- **ConnectX-8 NIC**: 800 Gb/s (~100 GB/s) per port (Blackwell era)
- **Quantum-2 switch**: 64 ports × 400 Gb/s

Typical big AI cluster: each HGX node has 8× 400 Gb/s InfiniBand NICs — one per GPU. Nodes connect through a fat-tree of switches. Any GPU in the cluster can talk to any other GPU with predictable low latency (~1 μs per hop).

InfiniBand also supports **RDMA** (Remote Direct Memory Access) and **GPUDirect RDMA** — GPU on node A can write directly to GPU memory on node B, bypassing both CPUs. Zero-copy, ultra-low-latency, essential for cross-node NCCL performance.

### The bandwidth cliff

Real numbers:

| Hop | Bandwidth | Latency |
|---|---|---|
| Same SM, register access | ~TB/s | ~1 cycle |
| HBM (same GPU) | ~3–8 TB/s | ~100 ns |
| NVLink (same NVSwitch domain) | ~900 GB/s – 1.8 TB/s | ~1 μs |
| InfiniBand (across nodes) | ~50–100 GB/s | ~2–5 μs |
| Ethernet | ~10–50 GB/s | ~10 μs |

Every step out costs ~10× bandwidth and ~10× latency. Great algorithms *stay local* as much as possible.

### Why interconnect changes what you can do

For inference:
- Model fits on 1 GPU → interconnect barely matters
- Model spans 8 NVLinked GPUs → NVLink bandwidth becomes THE speed limit
- Model spans multiple nodes → InfiniBand becomes THE speed limit

For training:
- Tensor parallelism (splitting single layers across GPUs) needs *insane* bandwidth → keep on same NVSwitch domain
- Pipeline parallelism (splitting layers between GPUs) tolerates lower bandwidth → can cross InfiniBand
- Data parallelism (whole model on each GPU, different data) only syncs gradients occasionally → InfiniBand fine

Detailed treatment: [[Parallelism]].

### Collective operations — the workload interconnect actually runs

Interconnect isn't just "one GPU sends a byte to another." Real workloads use **collectives**:

- **AllReduce** — every GPU has a vector; end state, every GPU has the sum of all vectors. Used for gradient sync in data parallelism.
- **AllGather** — every GPU has a piece; end state, every GPU has all pieces concatenated.
- **ReduceScatter** — inverse of AllGather.

Efficient implementations (ring-allreduce, tree-allreduce) minimize total bytes moved but stress the interconnect fabric. NCCL (see [[cuBLAS cuDNN and NCCL]]) implements all of these tuned per topology.

### Ethernet as a wildcard

Ethernet (specifically Ultra Ethernet Consortium and 800G Ethernet with RoCE) is trying to compete with InfiniBand for AI clusters. Cheaper, more vendors, but harder to get the low-latency deterministic behavior AI needs. Meta and some hyperscalers use it; NVIDIA prefers you buy their InfiniBand. Debate ongoing.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[DGX HGX and Superpods]] — the boards and racks these fabrics live on
- [[NVIDIA GPU Generations - A100 H100 H200 B200 GB200]] — each gen bumps NVLink; NVL72 changes the game
- [[GPU Memory Hierarchy - HBM L2 SRAM Registers]] — interconnect is the next tier below HBM
- [[cuBLAS cuDNN and NCCL]] — NCCL is the software that rides on this hardware
- [[Parallelism]] — this is why training strategies are shaped the way they are
- [[Inference Hardware]] — vendor-agnostic version

## 💡 So What
When comparing AI clusters, "GPU count" is half the story. Ask: how many GPUs are in one NVLink domain, and what's the InfiniBand fabric between domains? An 8× H100 with fast NVLink but slow node-to-node beats 32× H100 with slow interconnect on many workloads.

## ❓ Open Question
Ultra Ethernet + RoCE claims to close the gap on InfiniBand at lower cost, and Google's TPU pods use their own custom optical interconnect. Does NVIDIA's InfiniBand lock-in eventually crack, or does the software (NCCL topology awareness, GPUDirect) keep it sticky?

## 📚 Source
- NVIDIA NVLink and NVSwitch product briefs
- Mellanox / NVIDIA Quantum-2 InfiniBand switch datasheet
- NVIDIA GB200 NVL72 whitepaper (2024)
- "Bringing HPC Techniques to Deep Learning" — Andrew Gibiansky, 2017 (ring-allreduce origin)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
