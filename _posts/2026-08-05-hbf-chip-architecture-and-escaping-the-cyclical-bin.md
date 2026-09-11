---
layout: post
title: "HBF: What the Spec Says About Chip Architecture, and About Escaping the Cyclical Bin"
date: 2026-08-05 09:00:00
categories: [semiconductors, memory]
excerpt: "The first High Bandwidth Flash spec trades bandwidth and latency for 8× the capacity. What it can hold, why dataflow accelerators suit it better than GPUs, and why SanDisk and SK hynix want it for reasons that have little to do with each other."
---

*The first High Bandwidth Flash spec trades bandwidth and latency for 8× the capacity. What it can hold, why dataflow accelerators suit it better than GPUs, and why SanDisk and SK hynix want it for reasons that have little to do with each other.*

On August 3, ahead of FMS 2026, SanDisk and SK hynix released the first HBF technical specification through OCP (the Open Compute Project), just six months after the consortium formed in February. The two are the main contributors; Google and Tenstorrent joined during standardization, contributing mainly to technical validation and the standard itself.

*This is a translation of a piece originally published on Zhihu on August 5, 2026.*

### 1. Where HBF fits

As usual, the spec comparison first:

![Spec comparison, upper half](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/hbf-spec-table-top.jpg)

![Spec comparison, lower half](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/7856d115d73eb283eee55580dd33d018.jpg)

The difference between HBF and HBM is, at bottom, trading bandwidth and latency for capacity. One HBF stack is 512 GB; one HBM4 stack is at most 64 GB — an 8× capacity gap, and the only place HBF wins. The other two go the other way. HBM4 is already in production at 2.0–3.0 TB/s per stack, while HBF's Gen1 target is only 1.6 TB/s, and the three tiers defined in the spec start from 0.4. Latency differs even more: HBM4 is tens of nanoseconds, HBF about 10 microseconds — a hundred times.

SanDisk's own line on this is blunt: there are many ways to produce bandwidth; what matters is bandwidth, not latency. That explains why HBF works at all. As long as data can be prefetched ahead of time, 10 microseconds of latency can be hidden. The other problem comes from NAND itself: write endurance has a hard ceiling, unrelated to latency and impossible to route around. Stack the two constraints and HBF's boundary becomes clear. It can only take static, write-once-read-ten-thousand-times data like model weights; anything frequently rewritten still belongs to HBM.

Given those hardware parameters, the deployments follow. I tried to summarize them in a table:

![Where each kind of data lands: HBM, HBF, or SSD](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/f8566622bcfd0c9a757ba3709935f893.jpg)

The first three rows go to HBM, for the same reason. The KV cache during generation, activations, and gradients and optimizer state during training are all rewritten repeatedly. NAND's erase/write count has a physical ceiling, so these three kinds of data can't go on HBF. Activations have an extra reason: they're used the moment they're computed, with no prefetch window, and the 100× latency gap can't be dodged there.

The middle two (or three) rows go to HBF, and that's the reason it exists. Model weights and shared KV cache are written once and read many times, so the endurance limit doesn't bind. But read-only isn't enough; the other condition for landing on HBF is predictability. Inference executes layer by layer in order; when one layer finishes you know what the next will read, the data can be moved into a buffer ahead of time, and the 10 microseconds get covered. The 8× capacity gap is the gain; predictability is the precondition; both conditions must hold.

MoE is marked "unresolved" because it satisfies only the first condition. Expert weights are read-only and large, but which expert is activated is decided by the router at runtime; you don't know in advance what to read, and prefetch can't proceed (there are some expert-prefetching methods, of course).

The last two rows go to SSD. The RAG vector store and the model repository need bandwidth only intermittently — during retrieval or switching — and don't need to live on fast media. KV-cache overflow is more direct: eight HBF stacks are 4 TB, and the shared cache for a 10M context is 5.4 TB, which HBF itself can't hold.

### 2. The partners

Google and Tenstorrent joined during standardization, contributing mainly to technical validation and the standard. Google is one of the world's largest HBM buyers, and HBM has long been in short supply, with three vendors setting the price. An extra tier with "bandwidth close to HBM and 8–16× the capacity at similar cost," even without fully replacing it, dilutes dependence on HBM. Tenstorrent is RISC-V-based and chiplet-native, closely aligned with UCIe; the company is small and turns fast, and can bring out a genuinely HBF-equipped accelerator sooner than NVIDIA or AMD.

The core reason: the memory vendors can't set this standard alone; it needs tight co-design with the hardware, even with the model layer. At the August 6 "Breaking the Memory Wall with HBF" panel, for instance, Google sent Xiaoyu Ma, a senior engineer from DeepMind — the model side, not the hardware side. The signal is clear: the model side is defining what memory should look like. MoE models activating only a small fraction of the expert weights per token, long-context prefill, weights being read-mostly — these are exactly the patterns NAND can bear.

At the hardware level, the most important item in the spec is the xPU–HBF host interface, alongside electrical specs, packaging and reliability guidance for HBF stacks, and a software usage guide for reads and writes. That means HBF isn't a memory you plug in and use. The accelerator side has to implement the controller, the UCIe PHY, the area budget on the package substrate, and the memory-management logic itself.

Beyond that, at the architecture level, a thought struck me: dataflow accelerators suit the HBF route better than the GPU architecture does, and my guess is that the latency-hiding mechanisms are completely different. GPUs hide latency with massive multithreading — thousands of warps rotating under dynamic hardware scheduling. That mechanism is tuned for a few hundred nanoseconds of HBM latency; the register file and warp slots are sized to that latency budget. To hide 10 μs you'd need two orders of magnitude more work in flight, which doesn't hold up in area. Statically scheduled architectures like the TPU take another road: no cache in the traditional sense, the whole compute graph known at compile time, XLA moving data explicitly by DMA. The "stall" from latency isn't much of a problem; it's essentially a scheduling problem. Rough arithmetic: a 405B model at 8-bit weights is about 405 GB across 126 layers, about 3.2 GB per layer. Eight HBF stacks at 12.8 TB/s stream one layer's weights in about 250 μs. A NAND read latency is on the order of 10 μs — 4%. As long as you can prefetch layer N+1 while computing layer N, that latency vanishes completely. A Transformer's weight-read sequence in inference is fully deterministic; the prefetch window is absurdly large.

![SIMT vs. dataflow accelerator](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/b1019385164571ef843d2ca5e94d155b.jpg)

Finally, I think the industry isn't yet taking this seriously; it's more a route being tried. NVIDIA isn't in it, AMD isn't, Meta, Microsoft, and Amazon aren't, and neither are the other three NAND makers (Kioxia, Samsung, Micron). That says today's HBF is more like "one NAND camp plus one hyperscaler willing to validate plus one small vendor willing to go first" than an industry-wide consensus.

![Consortium membership](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/d46a7d2a6d9c27be6d804e0eed70d697.jpg)

### 3. Business and capital: the cyclical-stock angle

SanDisk's base business is consumer and enterprise NAND — the most commoditized, most brutally cyclical half of memory. Its valuation logic has always been "NAND cyclical." To shed the cyclical label, it wants, like the HBM route, to add customization and differentiation from peers. Today NAND's place in the AI narrative is marginal (large-capacity eSSD counts, barely). HBF is the first time NAND sits directly on the accelerator package, going from "the warehouse in the data center" to "part of the accelerator." The difference that makes to the valuation multiple is far bigger than the difference it makes to revenue.

SK hynix's position is the exact opposite; to me it looks more like a defensive play. If NAND squeezing into the memory hierarchy is going to happen sooner or later, it would rather lead it and hold a seat. It is also a NAND maker. SK hynix has its own NAND business (plus Solidigm), and that side's profitability has always trailed DRAM's by a wide margin. HBF is a channel for moving NAND capacity toward higher value, and SK hynix wants to stake out the track early.

More important, SK hynix really is trying on several fronts to escape cyclical-stock pricing: pushing MaaS at the group level, doing HBF proactively — the recent long-term contracts, the "memory as a service" pitch — all to convince capital that asset-heavy memory cyclicality isn't the whole story. And on the hardware side, **the packaging capability is the real moat; what the medium is doesn't matter. The barrier SK hynix has genuinely built up over these years is advanced packaging — TSV stacking, MR-MUF, warpage control — not the DRAM cell itself.**

![Media stacked into a memory hierarchy](/assets/images/posts/hbf-chip-architecture-and-escaping-the-cyclical-bin/3762d7898951b133166581e7b3fa439c.jpg)

So if there are this many media, could you stack them layer by layer into a complete memory hierarchy, then stop selling chips and sell a service instead, and jump the valuation from cyclical into some other category? Just a thought experiment, though.
