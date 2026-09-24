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

# DGX HGX and Superpods

Parent: [[NVIDIA Stack]]

## 🪝 The Hook
You can't just buy 8 GPUs and duct-tape them together. Power delivery, cooling, PCB trace lengths for NVLink, network cabling — get any of it wrong and your $300,000 of silicon runs at half speed. NVIDIA sells the *whole kitchen*, not just the chefs, and it comes in three shapes: **DGX**, **HGX**, and **Superpods**.

## ⚙️ Core Mechanism

### The three product tiers

| Product | What it is | Who buys it |
|---|---|---|
| **DGX** | A complete, NVIDIA-branded, NVIDIA-supported server — buy it, plug it in, done | Enterprises, research labs, "I want it to just work" |
| **HGX** | A reference *board design* — NVIDIA designs it, partners (Dell, Supermicro, Lenovo) build and sell variations | Cloud providers, hyperscalers, custom builders |
| **Superpod** | A blueprint for wiring many DGX/HGX nodes into one giant cluster | Hyperscalers, national labs, frontier AI labs |

Same underlying GPU silicon in all three. The difference is *integration and support*, not the chip itself.

### DGX — the appliance

A DGX system is a single chassis containing:
- 8 GPUs (DGX H100 = 8× H100; DGX B200 = 8× B200)
- NVSwitch fabric wiring all 8 together (see [[NVLink NVSwitch and InfiniBand]])
- Host CPUs (dual Intel/AMD, or on GB200, the on-package Grace CPU)
- NVMe storage, high-speed NICs, power supplies sized for ~10 kW
- NVIDIA's own tuned software image (drivers, CUDA, NCCL, container runtime — "DGX OS")

You buy it like a car — one SKU, one support contract, one throat to choke if something breaks. Expensive, but zero integration risk.

### HGX — the reference design others build on

**HGX** is NVIDIA's baseboard blueprint: "here's how to wire 8 GPUs + 4 NVSwitches on one PCB, here's the power/thermal envelope." NVIDIA licenses this design to partners (Dell, Supermicro, Foxconn, Quanta, and the hyperscalers themselves).

Those partners build the rest of the box around it — different chassis, different storage config, different networking choices — but the core 8-GPU NVLink domain is identical to what's in a DGX. This is how Azure, AWS, Google Cloud, and Oracle offer "H100 instances" without buying literal DGX boxes.

Kitchen version: DGX is a fully-equipped food truck NVIDIA sells you turnkey. HGX is NVIDIA handing out the blueprint for the "8-burner professional stove" so other truck builders can build their own trucks around it — same stove, different truck.

### Superpod — wiring many nodes into one cluster

A **Superpod** (DGX SuperPOD, or the newer GB200 NVL72-based pods) is a reference architecture for connecting many DGX/HGX nodes plus storage plus networking into one coherent, benchmarked, supportable cluster. It specifies:

- How many nodes (typically 32 to 256+ DGX nodes)
- InfiniBand fabric topology (fat-tree, rail-optimized)
- Storage layer (usually a parallel filesystem, e.g., for checkpoint I/O)
- Power and cooling layout
- Management software (cluster provisioning, job scheduling)

### The big shift: GB200 NVL72

Traditional SuperPODs wire together 8-GPU nodes over InfiniBand — fast, but a clear boundary at 8 GPUs (NVLink) vs slower cross-node (InfiniBand).

**GB200 NVL72** breaks that boundary. It's a *single rack* containing:
- 36 Grace CPUs + 72 Blackwell GPUs
- Wired with 5th-gen NVLink Switches at the rack level
- All 72 GPUs act as **one NVLink domain** — no InfiniBand hop needed within the rack
- ~130 TB/s total NVLink bandwidth, ~13.5 TB of unified HBM

That's a fundamentally bigger "single GPU-like" unit than the old 8-GPU limit. Multiple NVL72 racks then connect via InfiniBand into a Superpod, same as before — just with a much bigger building block.

### Why any of this matters for inference

- **Model doesn't fit on 1 GPU** → needs an NVLink domain (DGX/HGX node, or now a whole NVL72 rack)
- **Serving many concurrent users** → want multiple independent nodes, load-balanced (see [[Triton Inference Server]] and [[NIM Operator on Kubernetes]])
- **Huge models (1T+ params) at low latency** → need NVL72-scale domains so tensor parallelism doesn't cross a slow InfiniBand hop
- **Cost sensitivity** → HGX-based cloud instances are usually cheaper per GPU-hour than buying/running your own DGX

### Power and cooling — the boring part that decides everything

An 8× H100 DGX draws ~10 kW. A GB200 NVL72 rack draws **~120 kW** — more than 10 normal server racks combined, and it requires **liquid cooling** (air cooling can't remove heat that dense). This is why "just buy the GPUs" isn't how frontier compute works — you also need a data center retrofitted for liquid-cooled, 120kW-per-rack density, which very few facilities had before 2024.

This physical reality is a real bottleneck on how fast the whole industry can scale up, independent of chip supply.

## 🔗 Connects To
- [[NVIDIA Stack]]
- [[NVLink NVSwitch and InfiniBand]] — the fabric these systems are built from
- [[NVIDIA GPU Generations - A100 H100 H200 B200 GB200]] — GB200 is the chip; NVL72 is the rack
- [[Parallelism]] — cluster shape directly determines which parallelism strategies are viable
- [[Inference Hardware]] — vendor-agnostic version of "what hardware do I need"

## 💡 So What
When evaluating "can we run model X," the real question often isn't "do we have enough GPUs" — it's "do we have enough GPUs *in one NVLink domain*, with enough power and cooling." Cluster topology and data center physics are as much a constraint as raw GPU count.

## ❓ Open Question
Liquid cooling at 120kW/rack is a huge operational shift for most data centers. Is the industry retrofitting fast enough, or does power/cooling infrastructure become the actual bottleneck on AI scaling before chip supply does?

## 📚 Source
- NVIDIA DGX H100 / DGX B200 datasheets
- NVIDIA HGX platform product briefs
- NVIDIA DGX SuperPOD reference architecture whitepaper
- NVIDIA GB200 NVL72 whitepaper (2024)

---
*Status key: 🌱 seedling → 🌿 growing → 🌳 evergreen*
