---
layout: post
title: "From OpenAI's Chip to the Endgame of In-House LLM Silicon"
date: 2026-06-27
categories: [hardware, industry]
excerpt: "Network design has shifted from capability-driven to efficiency-driven, and efficiency optimization is sinking from the system level into the hardware. When the last gains have to be squeezed out of silicon, the model, the system, and the chip end up designed by the same hands."
---

*Network design has shifted from capability-driven to efficiency-driven, and efficiency optimization is sinking from the system level into the hardware. When the last gains have to be squeezed out of silicon, the model, the system, and the chip end up designed by the same hands.*

In June 2026, OpenAI announced its first chip, codenamed Jalapeño.

*This is a translation of a piece originally published on Zhihu on June 27, 2026 — before OpenAI's Hot Chips talk. The architectural details that came out later are in my Jalapeño piece.*

Tom's Hardware estimates the compute die at about 25.46 × 33 mm ≈ 840 mm², just under the EUV reticle limit (about 858 mm²) — one large monolithic compute die, rather than the multi-die approach NVIDIA and AMD take on their training chips. The repeated columnar layout is consistent with a range of tiled AI-accelerator architectures. Given how short the manufacturing cycle was, and that the people and the team were poached from Google, my guess is it's probably not far from a TPU.

But information is badly lacking. Most of Jalapeño's hardware details are analysts' estimates from one low-resolution image; OpenAI and Broadcom have explicitly declined to disclose specs, so there isn't much microarchitecture to analyze. The few performance claims — "about 50% lower cost per token" and "on par with Blackwell" — are self-reported. Still, I think there's a lot of information behind the product, and I'll use it below as the example for how the hyperscalers' chip-building is coupling with the direction of large-model development.

My thesis: network-architecture design has shifted from capability-driven to efficiency-driven, and efficiency optimization is sinking from the system level (MLSys) to the hardware level. When efficiency can only be squeezed further at the hardware layer, the network architecture, the system, and the chip need to be designed jointly by one party. OpenAI's first chip, Jalapeño, is one instance of this trend, and this piece uses it as the thread.

## 1. Design goal: from accuracy-first to efficiency-first

For the first years after the Transformer, network-architecture iteration was basically capability-driven: pile on parameters, go deeper and wider, improve attention, all for lower loss and higher benchmark scores. The implicit premise was accuracy first, cost second.

Goldman Sachs' 2026 research points out that compute has become the binding constraint on scaling AI. With inference demand growing faster than available capacity, the key differentiator is no longer model quality alone but the ability to acquire and finance compute more efficiently and reliably.

The first question MLA, MoE, and sparse attention have to answer today is no longer "can it be more accurate" but "can it cut per-token cost and raise throughput without losing accuracy." MLA's purpose is to cut the KV cache to a fraction of its size with almost no accuracy loss; MoE's is to keep active parameters far below total parameters and so lower per-token compute; sparse attention's is to bring attention cost down from quadratic in long contexts. What they share: accuracy holds the baseline, and the variable being optimized is efficiency.

Academia has formalized this too. Token efficiency is now a formal metric — how many correct answers per thousand tokens. OckBench's data shows that on the same problem, several models with similar accuracy can differ by 5× in token consumption. With capability held constant, differences in output length alone can move cost by several times.

Epoch AI gives a more precise longitudinal comparison. To reach about 27% accuracy on FrontierMath, o4-mini (high reasoning effort) in April 2025 needed about 43 million output tokens; by December 2025, GPT-5.2 (low reasoning effort) needed only about 5 million — an 8.6× drop in output tokens in eight months.

There's a spot here that is easy to misread. Epoch, in the same article, explains that once GPT-5.2's higher per-token price is factored in, actual cost fell only about 3×, not 9×. Tokens fell about 9×, cost only about 3×, because the new model is more expensive per token. Gains in token efficiency don't pass through to cost proportionally.

There is also a trend in the opposite direction to account for: test-time compute. At a fixed capability point, models use fewer and fewer tokens to complete the same task; but frontier models are spending *more* tokens, using longer reasoning chains and RL scaling to push the capability ceiling up — Epoch's article is itself about the scaling of inference compute. So tokens at a fixed capability point are falling while the frontier pushes the capability point up by spending more tokens; both trends exist at once. Cutting cost and burning compute coexist.

## 2. Total tokens are growing fast

Tokens per fixed capability point are falling, but total token volume is growing several-fold per year.

Goldman Sachs' May 2026 research, *Decoding the Agentic Economy* (analyst Jim Schneider), puts the scale at 24× growth in token consumption between 2026 and 2030 as consumers and enterprises adopt AI agents, reaching 120 quadrillion tokens per month.

Google's data corroborates the curve. By Pichai's account, the company's monthly token processing went from about 9.7 trillion in April 2024, to about 480 trillion at I/O in May 2025, to nearly 1.3 quadrillion in Q3 2025 (around October), to 3.2 quadrillion per month by I/O 2026 — about 7× in the past year.

When volume grows several-fold a year, any optimization in per-token cost is amplified into a large difference in spend. That's the chip vendors' motive for cutting cost. Goldman estimates semiconductor vendors are cutting per-token inference cost by 60–70% a year, and expects this to bring a positive gross-margin inflection, roughly around the first half of 2026.

The same data has another reading, though. Goldman's own conclusion leans optimistic: with unit cost falling 60–70% a year, the cumulative drop over a few years is large, so even with volume up 24× the total bill needn't explode — it may even help gross margins. So "the denominator is ballooning, therefore cost must be cut" and "cost-down plus volume-up brings a margin inflection" are two readings of the same dataset. For a player building its own chips — OpenAI, say — the logic isn't only saving money. It's controlling its own cost curve, making each token cheaper at its own scale, while reducing its dependence on NVIDIA.

## 3. Today's efficiency optimization mostly happens at the system level

This efficiency optimization currently happens mainly at the system level. MLA addresses the KV-cache memory pressure that inference-serving systems expose; MoE's expert partitioning has to coordinate with expert parallelism and communication in the distributed system; sparse attention has to accommodate the operator implementations in the software libraries.

That is, the network architecture is currently co-optimizing efficiency with the software deployment system. It treats the chip as a given black box and redesigns the architecture on top of that box to squeeze per-token efficiency. This is the efficiency paradigm of the MLSys era.

But it is an intermediate state. When the efficiency the system level can squeeze tops out, efficiency optimization will keep sinking toward the hardware level. Network architecture will shift from adapting to the software stack to interfacing directly with the chip's physical characteristics — even asking the chip to change for a given structure.

## 4. Two kinds of system-level co-design: Xiaomi's SWA and DeepSeek's NSA

Xiaomi and DeepSeek both use sparse attention to cut long-context compute, but their sparse operators go in different directions, and their system co-design differs accordingly.

Xiaomi's MiMo uses sliding-window attention (SWA), a static structured sparsity. Each token attends only to the W tokens in a fixed window before it (MiMo takes W = 128), with a few global-attention layers to handle long-range dependencies. Its key property is that the sparsity pattern is fixed at compile time: which positions to compute and which to skip is independent of the input data, and fully predictable. That property is good for the system. Operators can be statically tiled; the KV cache can be a rotating buffer, with old tokens overwritten as they slide out of the window; the memory footprint is constant rather than linear in sequence length; memory access is regular and contiguous, with no branches or dynamic addressing. The compute SWA saves is handed over in a form the system can arrange in advance; its style of co-design is to make the operator fit the regular access patterns the system is good at. The price is limited expressiveness: a fixed window means exact dependencies beyond W can only be passed indirectly through stacked layers — a weakness for tasks that need precise long-range recall.

DeepSeek's NSA (Native Sparse Attention) is dynamic, learnable sparsity. It has three parallel branches — coarse-grained token compression, fine-grained token selection, and a sliding window — merged by a learned gate. The key is the token-selection branch. Which keys each query attends to is computed by the network at runtime, depending on the input. That shifts the complexity onto the system. The sparsity pattern is no longer predictable; the operator has to handle dynamic gathers (fetching KV by indices computed at runtime), a variable number of effective KV entries, and load imbalance between queries. A naive implementation would, because of these irregular accesses, lose in memory latency the FLOPs it saved in theory. So NSA can't be a change at the network-architecture level alone; it has to be designed together with a hardware-aligned custom kernel — organizing memory loads by GQA/MQA group so one loaded KV block is shared by all heads in the group, and aligning access granularity to the Tensor Core and to GPU memory transactions — to turn dynamic sparsity's theoretical gain into an actual decode-stage speedup. Its style of co-design is the opposite of SWA's: not the operator adapting to the system, but the operator and its kernel implementation bound together and designed jointly.

Compare the two and the difference is clear. SWA starts with a regular sparse structure, which the system then executes efficiently. NSA starts by wanting data-dependent sparse expressiveness, then requires the operator and kernel to absorb the resulting irregularity. The former pushes the co-design difficulty to compile time, adding almost no runtime burden, at the cost of expressiveness. The latter buys stronger expressiveness, lifts the co-design difficulty to runtime, and solves it with a custom kernel. These are the two typical modes of system-level co-design: one has the network architecture actively converge to a system-friendly form; the other binds the network architecture and the system implementation deeply and designs them together.

NSA's approach already points to the next layer down. When an operator's gains can only be realized by aligning with Tensor Cores and memory transactions, it is in effect already making demands of the hardware's physical characteristics. It still sits at the system level, solving the problem with a kernel — but the language it uses to solve it is the hardware's.

## 5. Next: direct co-design with the hardware microarchitecture

My judgment, then, is that the next step for network architecture is direct co-design with the hardware microarchitecture. All the co-design discussed above happens at the system level: the designer treats the chip as a given, unmodifiable object and squeezes efficiency on top of it with operators and kernels. Operators like NSA already show the system level's limits. When a sparsity pattern's gains depend heavily on specific hardware primitives — the Tensor Core's shape, the granularity of memory transactions, the size of on-chip cache — continuing to align through kernels is just accommodating a chip that wasn't designed for it, and the space to squeeze gradually tops out. The logical next step is to turn it around and have the hardware change for the network architecture.

There's a contradiction to handle here. A chip is a one-to-two-year cycle with high fixed cost; network architectures turn over a generation every few months — Jalapeño's nine-month tape-out, discussed below, was reported as a fast case for the industry. Redoing a chip for one architecture, while the architecture keeps changing, is a conflict.

My view is that the way out isn't redoing the chip for a specific architecture but choosing the level at which to co-design. What's worth hardening into hardware is the primitives that are stable across architectures, not some whole architecture that will go obsolete. The demands DeepSeek made of chip vendors in its ISCA '25 hardware-reflection paper, *Insights into DeepSeek-V3* — FP8 accumulation precision, native fine-grained quantization, silicon photonics or low-latency inter-node interconnect — are basically all primitive-level and cross-model, not a chip built specifically for MoE. What sinks into hardware, in other words, is not the chip taking the shape of some architecture, but the chip providing the primitives a class of architectures needs most. That captures the dividend of hardware-level co-design without being locked to a single architecture's iteration.

Cerebras is the extreme case on this road. The wafer-scale engine makes the whole wafer one chip, trading huge on-chip SRAM and on-chip interconnect for low latency — essentially hardware extremely specialized for one workload shape: low latency, weights resident. But its situation also illustrates the contradiction above: the more thoroughly you specialize, the more you depend on one class of architecture and model staying dominant, and the less commercial flexibility you have. Primitive-level co-design is looking for the balance between the efficiency of specialization and the flexibility of generality.

## 6. Jalapeño

Put the logic above onto a concrete case: OpenAI's first chip, Jalapeño, announced in June 2026 in partnership with Broadcom and Celestica. Its positioning matters more than any specific performance number: the model, the kernels, the serving system, and the chip are jointly designed by one party. In OpenAI's words, the optimization spans chip architecture, kernels, memory system, network, scheduling, deployment, and product experience, with every layer built around the same goal — making its own models faster, cheaper, and more reliable. That is the joint design described above.

Hardware detail on Jalapeño is scarce. OpenAI and Broadcom declined to disclose specs; most details are analysts' estimates from a low-resolution image. A few things can be said.

Per Tom's Hardware, the compute die is about 25.46 × 33 mm, about 840 mm², just under the EUV reticle limit of about 858 mm². "Monolithic" here means the compute portion uses one large die. The package actually holds one large compute chiplet plus one I/O chiplet, plus two structural dummy dies, plus six HBM stacks; it isn't a monolithic SoC, but it genuinely doesn't split compute across several dies the way NVIDIA's and AMD's training chips do. Choosing a large compute die over multi-die is usually about pushing latency to the minimum, consistent with its positioning toward low-latency reasoning and agentic serving.

It uses HBM rather than cheaper DRAM — also a choice for low-latency inference, not for stacking cheap capacity.

The repeated columnar layout in the image is consistent with various tiled AI-accelerator architectures, and Tom's also judges that it looks like a Broadcom-style systolic-array template. Given the very short manufacturing cycle and a team poached from Google, it's probably not far from a TPU in approach — a judgment Tom's supports — though this is still inferred from one image.

Performance claims need attributing. OpenAI officially says only that performance per watt is significantly better than current state-of-the-art hardware, with no specific numbers or benchmarks. The frequently cited "on par with Blackwell" or "can beat MI350" is mostly analysts' inference (Tom's, for example), and the same analysis adds that against NVIDIA's Rubin and AMD's MI400 it's hard to say. As for numbers like "about 50% lower cost per token," I couldn't find a source in OpenAI's official material. There's a 50× efficiency / 35× cost figure circulating, but that's about GPT-5.5 Codex running on NVIDIA Blackwell, not Jalapeño — cite with care.

Analysts describe Jalapeño as a captive cost-optimization play: not for sale, not aimed at public leaderboards, serving only OpenAI's own models. Its goal is direct — make every token ChatGPT generates (about 900 million weekly actives) cheaper, while reducing dependence on NVIDIA.

## 7. AI in the chip's design

Finally, a related point. What's notable about Jalapeño isn't only the specs; it's two numbers.

First, OpenAI's in-house chip team is only a few dozen people — a core of about forty, reportedly — where a normal advanced ASIC design often takes hundreds or even thousands of engineers.

Second, this is its first time building a chip, and it took only about nine months from design to tape-out, where the industry usually needs a year and a half to two years.

OpenAI also says engineering samples have already run real inference tasks in the lab, including GPT-5.3-Codex-Spark, at production-target frequency and power. Behind this stands Broadcom's mature design platform and a large stock of ready IP. But even counting Broadcom's contribution, a team of a few dozen producing a usable result this fast on its first chip says that frontier models can now provide real help in complex engineering domains. President Greg Brockman said the degree to which the models accelerated the tape-out surprised even them.

That's consistent with this piece's thread. Model, kernels, and chip jointly designed by one party — and the designers now include the model itself. The same model is both the thing being served and a participant in designing the hardware that carries it. It's the first verifiable example of OpenAI migrating its proven coding ability into a professional engineering domain.

My earlier argument was that hardware microarchitecture and network architecture advance together, but tape-out cycles are usually long; whether AI assistance ends up bringing that goal closer, I don't know yet.

*Postscript on Cerebras: ouch, that stock price. Glad I didn't buy.*
