---
layout: post
title: "LLM Decode Accelerators, Part 1: The Cerebras Wafer-Scale Engine"
date: 2026-04-12
categories: [hardware, llm]
excerpt: "Reticle stitching, yield, and heat shape everything above them. Why the WSE-3 is 900,000 tiny cores instead of a giant systolic array, what MemoryX and SwarmX actually do, and how it runs GEMV."
---

*Reticle stitching, yield, and heat shape everything above them. Why the WSE-3 is 900,000 tiny cores instead of a giant systolic array, what MemoryX and SwarmX actually do, and how it runs GEMV.*

*This is a translation of a piece originally published on Zhihu on April 12, 2026.*

## 1. Process

The key to wafer-scale computing isn't "make the chip bigger." It's how, under existing semiconductor manufacturing constraints, you turn a whole wafer — which was never supposed to be one chip — into one working chip. The first constraint is the lithography reticle limit: a modern exposure system's single-shot field is about 26 mm × 33 mm, so a wafer-scale design has to use field stitching across reticles rather than the conventional one-field-per-chip approach. The second is yield: once the area grows to a whole wafer, manufacturing defects stop being occasional and become certain, so the architecture has to natively support fine-grained redundancy and fault bypass. The third is packaging and system: a chip this large faces power-delivery, test, and cooling pressures far beyond a normal processor's. Which is why wafer-scale was never simply "a bigger die"; it's the result of co-optimizing manufacturing, architecture, and packaging.

On that process foundation, Cerebras isn't just making bigger chips generation after generation; it's gradually expanding the single giant wafer chip into a complete system-level compute platform. From the CS-1/CS-2 era, the WSE was first used for large-model pre-training, continued pre-training, and fine-tuning. Now, with LLMs everywhere, Cerebras's official inference page stresses production-scale inference, very high tokens/s, and an instant-response experience; when Cerebras Inference launched in 2024 it led with high output speed on Llama 3.1 8B and 70B, packaged as "the world's fastest AI inference."

From public data, I first compared several products' compute throughput and SRAM capacity per unit area, to show the strength of the wafer process:

![Compute throughput and SRAM per unit area, several products compared. These baselines were chosen because they're the only ones with public area data.](/assets/images/posts/llm-decode-accelerators-1-cerebras-wafer-scale-engine/04d5d27bafecb740ed818e7449550f7f.jpg)

Of course, the area advantage is tied to the architecture too. A GPU has to spend a lot of area on HBM controllers, PHYs, off-chip high-speed I/O, a large cache hierarchy, complex control, and general-purpose execution support; the WSE's design goal is more focused, so more area goes directly to compute units + local SRAM + a simple, regular interconnect. So it isn't that "transistor density is higher" but that "the share of area actually allocated to the useful AI datapath is higher." The cost is that a lot of complexity is shifted into the programming model and dataflow organization, and heat, power delivery, packaging, and test are all harder — small compute units failing, and so on, is routine.

## 2. Performance analysis

For a giant monolithic chip like this, external I/O, packaging, power delivery, and cooling are all harder to scale, so it doesn't suit the conventional processor's route of "keep piling on off-chip bandwidth." The more natural direction is to keep more data reuse on-chip, using higher on-chip SRAM bandwidth, on-chip fabric bandwidth, and matching dataflow and scheduling to reduce dependence on off-chip bandwidth. So the traditional compute-bound / memory-bound dichotomy, governed by off-chip DRAM bandwidth, is clearly weakened; with die area, on-chip SRAM capacity, and on-chip bandwidth all vastly enlarged, the system bottleneck shifts to whether compute can be fed adequately and how data communicates efficiently on-chip and across systems — so the main trade-off is closer to compute-bound versus communication-bound.

![Specs by product](/assets/images/posts/llm-decode-accelerators-1-cerebras-wafer-scale-engine/d9ead7db6c3d94c726d217ad1479b16f.jpg)

The parameters agree. The evolution from WSE-1 to WSE-3 has been a continuous piling-up of compute resources, on-chip SRAM, and on-chip interconnect capability on whole-wafer area. WSE-1 already offered 400,000 AI cores and 18 GB of on-chip SRAM; WSE-2 grew to 850,000 cores and 40 GB; WSE-3 reaches 900,000 cores, 44 GB of on-chip SRAM, and 125 PFLOPS of peak compute. Meanwhile Cerebras has long stressed extreme on-chip and fabric bandwidth: WSE-1 at 9 PB/s of on-chip memory bandwidth and 100 Pb/s of fabric, WSE-2 up to 20 PB/s and 220 Pb/s, and WSE-3, per the current product page, on the order of 21 PB/s and 214 Pb/s.

Because of the huge on-chip memory, they designed a method called weight streaming that decouples parameter capacity from on-chip storage capacity, setting it apart from typical GPU/TPU scheduling with a more flexible compute–memory schedule. On a conventional GPU, compute and memory are bound together: however many GPUs you have is however much total memory you have, and as soon as the model exceeds one card's memory by a hair, you have to add more GPUs and bring in complex distributed mechanisms like model parallelism and pipeline parallelism. Cerebras's own comparison: if a GPU has 80 GB and the model needs 82 GB, you add GPUs; in the weight-streaming architecture the parameters live in external MemoryX, whose capacity scales independently, so you're not forced to add a pile of compute chips just because you "want a bit more parameter capacity." Compute sits on the WSE, parameters in MemoryX, and the two scale separately.

Regarding weight streaming, GPUs have similar scheduling in software, so the real difference is in the hardware design of MemoryX. On external memory, Cerebras's official blog describes MemoryX as a "dedicated, external memory device" and states explicitly that it "uses flash and DRAM," without giving the DRAM generation or the die/channel configuration. MemoryX provides *central weight storage* — the model parameters kept in an independent memory system and streamed efficiently to the CS-2/CS-3; the CS-3 can be configured with up to 1,200 TB of external memory. From the limited official documentation one can infer that MemoryX is more like a tiered storage + scheduling controller system: flash as the high-capacity weight-persistence layer, and DRAM most likely serving as the near-end buffer / staging / reordering / streaming queue, preparing the layer weights about to enter the WSE ahead of time.

![MemoryX (Cerebras)](/assets/images/posts/llm-decode-accelerators-1-cerebras-wafer-scale-engine/68cfda7efecb4afe1ee8743d21b96f56.jpg)

To fully decouple parameter capacity from on-chip storage, SwarmX needs introducing too. Compared with NVLink, a more general high-bandwidth GPU/CPU interconnect, SwarmX is a broadcast/reduce fabric designed specifically for Cerebras's weight-streaming training. It's highly specialized, handling mainly the downward broadcast of layer weights and the upward reduction of gradients. NVLink is aimed at more general GPU remote-memory access, tensor-parallel traffic, and the assorted exchanges — activations, parameters, KV, collectives — between GPUs.

My first thought here was that a specialized interconnect lacks flexibility and MoE mapping might not work — but then it occurred to me that they do inference on a single wafer anyway, so I looked into the on-chip interconnect as well. Its flexibility isn't unlimited all-to-all, but if MoE can be organized as a localized dispatch problem, the on-chip interconnect can handle it; academia and industry already have plenty of such methods.

## 3. Microarchitecture

![Borrowed from EPCC](/assets/images/posts/llm-decode-accelerators-1-cerebras-wafer-scale-engine/f2223212a842980de75ced1da3684f71.jpg)

Finally, the compute microarchitecture. One WSE-3 core consists of:

- 48 kB of SRAM per core
- 512 B of local cache per core
- 8-way SIMD for 16-bit data (FP/BF16)
- 16-way SIMD for 8-bit data (fixed-point/INT8)

Inside each die is a 2D mesh fabric; the interconnect extends across reticle/die boundaries at full performance; hardware redundancy is built in to route around failed locations; software sees a single unified 2D mesh. The WSE-3 isn't the conventional GPU/TPU organization of a few large cores plus a shared cache/HBM, but a vast number of small cores, each with private SRAM, under unified on-chip dataflow scheduling. At first I found this odd, because I'd have expected a larger systolic-array-style accelerator to strengthen data reuse and make better use of the rich on-chip storage.

I later guessed a few reasons. First, Cerebras cares a lot about nonlinear ops, small operators, and sparsity, and didn't want to tie the architecture to a matrix core; Cerebras officially says the goal from the start was a core suited to fine-grained, dynamic sparsity in neural networks. Academics have also used it for scientific computing like stencils. Second, heat: the WSE-3 is a 46,255 mm², roughly 4-trillion-transistor wafer-scale chip with very high power density per system. Cerebras specifically stresses that the engine block delivers power "straight into the face of the wafer" because conventional packaging can't reach the required power density; likewise, conventional air cooling can hardly achieve sufficiently uniform, low-thermal-resistance cooling over such a large single piece of silicon. High-density compute units like systolic arrays readily create hot spots. Third, yield: as noted, the wafer readily has bad spots, and if the array is too large and too concentrated, defects can't be routed around, and fault tolerance and remapping are usually harder.

Finally, the actual inference setup. Although Amazon has already tried using the WSE as a decode-only accelerator (rich indeed — no wonder the stock has been surging), currently prefill and decode still run together for end-to-end speedup; otherwise prefill's KV-cache transfers would take time too. Per Wafer-LLM, GEMM corresponds to prefill and GEMV to decode (side grumble: running one batch at a time on a chip this expensive is wasteful — which is presumably why Amazon runs multi-batch decode on it). GEMM is cut along both dimensions and uses a compute-shift loop, computing while shifting. GEMV: cut along one dimension and replicate along the other — the L dimension is replicated along the x-axis — and local compute + all-reduce use the high on-chip communication bandwidth to hide the shortage of off-chip bandwidth, relieving the memory-bound problem.
