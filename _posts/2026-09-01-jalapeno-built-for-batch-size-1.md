---
layout: post
title: "Why OpenAI's Jalapeño Is Built for Batch Size 1"
date: 2026-09-01
categories: [hardware, llm]
excerpt: "Agentic workloads pin inference near concurrency 1. Every odd design choice in the chip — out-of-order cores, an L1 cache, a flattened memory hierarchy — falls out of that constraint. Then there's the part where Codex writes the kernels."
---

*Agentic workloads pin inference near concurrency 1. Every odd design choice in the chip — out-of-order cores, an L1 cache, a flattened memory hierarchy — falls out of that constraint. Then there's the part where Codex writes the kernels.*

I've been following OpenAI's and Cerebras's custom silicon for a while and have written about both before (in Chinese: [From OpenAI's chip to the endgame of in-house LLM silicon](https://zhuanlan.zhihu.com/p/2054021097989973298), [Cerebras fundamentals, part 2](https://zhuanlan.zhihu.com/p/2038482579343586667)).

Most of the coverage of Jalapeño since Hot Chips has been about the technical specs. I want to add the demand side — what the workload actually forces — and map that onto the specific design decisions. The second half looks at the other thing that's new here: how much of the hardware and the software was designed by AI, and what that implies.

*This is a translation of a piece I originally published on Zhihu on September 1, 2026.*

## The constraint: batch can only go a few multiples

The first source of pressure is the agentic side — which is also where model vendors' revenue now mostly comes from. A coding-agent session reads code, edits files, runs tests, reads the errors, and goes again; context accumulates across turns and is almost never released mid-session. OpenAI sized the headroom in its scale-up network for 2–4 million tokens of context. At GQA-8 and 320 KB of KV per token (a 70B-class number; bigger models only go up), a single 2-million-token session is 610 GB of KV cache alone. One agentic session's KV is 2.6× the entire HBM on one chip.

Second, models keep growing, so the weights' share only rises. Jalapeño carries 216 GiB of HBM4 per chip, about 232 GB. A trillion-parameter-class MoE is around 500 GB of weights even stored in MXFP4 — it doesn't fit on one chip, so it has to be sharded across several. After sharding, each chip's HBM splits in two: one part holds its slice of the weights; whatever's left is the KV-cache budget.

So batch size isn't something you get to choose. Agentic long context pushes each sequence's footprint up by one to two orders of magnitude; in practice, the number of sequences a chip group is serving at once is single digits, and the limit is 1. That's concurrency 1. Everything below assumes that limit point; I'll use "low batch" for the whole region where fixed overheads can no longer be amortized. The choice of metrics confirms it.

## The metrics say the same thing

At Hot Chips 2026, OpenAI said the design was organized around **time to last token** and **tokens per joule**, and explicitly dropped chip count, per-chip throughput, and **time to first token (TTFT)** as framings.

TTFT has mattered for so long on one premise: a human is reading the stream, and a fast first token makes the system feel responsive. (I still personally prefer a low TTFT — it makes the model feel like it's thinking.) But nobody reads the intermediate stream inside an agent loop; only the complete output gets fed to the next step. Switch to TTLT and the reader is no longer a person. It's the next model call.

The way the bandwidth ceiling is stated is the second signal. OpenAI says 128 chips have over 1 PB/s of aggregate bandwidth, which as a pure bandwidth bound works out to 1,000–2,000 tokens/s/user without speculative decoding and 5,000–10,000 with it. Note the unit: per user.

The third signal was the live demo: Codex CLI on an internal model at 1.2 ms TPOT (time per output token); the summary slide claims sub-millisecond token-to-token latency on frontier models. Sub-millisecond TPOT only means anything if you intend to chain thousands of tokens serially.

![TTFT covers prefill, TPOT is one decode step, TTLT is the whole chain. A human reader cares about the first; the next model call only cares about the last.](/assets/images/posts/jalapeno-built-for-batch-size-1/01-ttft-tpot-ttlt.png)

**What breaks at low batch**

Kernel launch, synchronization barriers, memory latency, cross-core collectives — the absolute cost of these fixed overheads doesn't change with batch. At batch 256 they're spread across 256 units of work; at batch 1, not at all.

A matrix-vector multiply with M=1 touches each weight byte for exactly one multiply-accumulate: arithmetic intensity around 1 FLOP/byte, against a roofline knee in the hundreds for an HBM4 system. This workload has formally gone back from compute-bound to memory-bound.

---

## Only one kind of silicon eats this operating point

OpenAI's compute map is the biggest table in public: 10 GW from NVIDIA, 6 GW of AMD MI450, 10 GW of custom silicon with Broadcom, 4.5 GW from Oracle, plus odds and ends — about 33 GW in total.

Filter that table by architecture instead of vendor and it collapses:

- **SIMT (NVIDIA, AMD)** hides latency with occupancy; at concurrency 1 there's no other warp to switch to.
- **Systolic array + scratchpad (Google TPU, AWS Trainium)** relies on shapes known at compile time and tiles big enough to keep a static DMA pipeline full; at concurrency 1 the pipeline can't fill, and on a very large systolic array only one row of silicon is doing anything at M=1.
- **Wafer-scale SRAM (Cerebras)** takes the weights out of HBM and keeps them resident on-chip, so the memory latency doesn't exist in the first place. Structurally, this is the only road that fits the operating point.

![Inference hardware on the latency–throughput Pareto curve. Jalapeño sits with the interactivity-first camp, but not at its edge.](/assets/images/posts/jalapeno-built-for-batch-size-1/02-pareto-quadrant.png)

Which is why, in a pile of gigawatt-scale contracts, a 750 MW wafer-scale deal suddenly shows up. The agreement signed on January 14, 2026 was described by OpenAI as adding "ultra-low-latency" compute to the platform, with the use case pointed squarely at latency-sensitive work like coding agents and voice; Cerebras's own number is up to 15× faster than GPU systems on the same LLMs (vendor claim; I haven't seen a third-party reproduction). This isn't routine supplier diversification. It's filling an architectural hole for one class of workload.

## But there just isn't enough of it

In scale, Cerebras is a rounding error. The base commitment is 750 MW, with options to reach 2 GW by 2030. Against a 33 GW total, that's under three percent. For comparison, the Broadcom custom-silicon plan is 10 GW.

**Capacity is a hard constraint.** Cerebras is sitting on roughly $25 billion of backlog, and Feldman has admitted in interviews that they can't build fast enough — he blames data-center construction not keeping pace. The more structural problem is process lock-in: stitching multiple reticles on one wafer into a single chip is a process deeply tied to TSMC, not something you fix by switching to a foundry with spare capacity. At roughly 25 kW per CS-3 node, 750 MW is on the order of thirty thousand systems, and that volume alone strains TSMC's wafer-scale capacity.

**Economically, it's the most expensive way to buy latency.** Weights resident in SRAM means trading enormous silicon area for low latency; the per-token cost structure is inherently high. And SRAM capacity is finite — agentic workloads want low latency *and* a large KV cache at the same time, and multi-turn long-context KV keeps growing across turns, which lands exactly on the wafer-scale design's thinnest spot. Cerebras itself concedes it can't take prefill: its disaggregated-inference scheme with AMD promises up to 5× throughput and ships in Q4 2026; on AWS, Trainium3 does prefill and CS-3 does decode.

OpenAI isn't the only one who reached this conclusion, incidentally. At the end of December 2025, NVIDIA paid about $20 billion for Groq's technology license and core team; three months later Groq 3 LPX shipped as an NVIDIA product. Jensen's deployment plan at GTC: of the data-center footprint allotted to coding applications, a quarter goes to Groq chips and the rest to Vera Rubin. "Low-latency inference needs a different kind of silicon" is already industry consensus; the disagreement is only over how many kinds of silicon to cover it with.

---

## So, a third road

Lay the constraints out: be competitive at concurrency 1; keep the HBM tier of capacity, or you can't hold agentic long context; have a cost structure that supports gigawatt-scale mainline deployment, not specialty-part pricing.

Cerebras's road satisfies the first and gives up the other two. SIMT and systolic arrays satisfy the last two and give up the first.

Jalapeño wants all three. Its method copies neither side; it's a third one: **make the memory latency itself short, and absorb the rest dynamically in the microarchitecture, rather than amortizing it with piles of parallel work.**

![Three ways to live with memory latency: absorb it dynamically (OoO + L1), hide it with a static pipeline (scratchpad + DMA), or hide it with concurrency (SIMT).](/assets/images/posts/jalapeno-built-for-batch-size-1/03-three-memory-hierarchies.png)

The three differ in what they trade away:

**SIMT trades concurrency.** The latency isn't shortened, it's tolerated — when a warp stalls, the scheduler switches to the next ready one. To cover 400–600 cycles of HBM latency you need "N warps × latency" worth of work in flight, which means high occupancy. So it naturally prefers big batches and big shapes. Decode has low concurrency to begin with, and utilization on this path collapses. Fixed costs like launch latency and barrier latency can likewise only be amortized by each core doing more work.

**Scratchpad + DMA trades predictability.** Latency is digested by compile-time software pipelining: DMA moves block N+1 while block N computes, and in steady state compute never waits. The price is that shapes must be known in advance, access patterns must be statically analyzable, and tiles must be big enough to cover the fixed cost of DMA startup plus inter-stage barriers. Data-dependent access breaks it — MoE expert routing, sparse KV gathers: you can't prefetch an address that hasn't been computed yet. TPU and Trainium both take this road.

**Out-of-order + L1 moves both ends.** First, make the latency itself short: a core slice sees only its own slice of HBM, the hierarchy is flattened to the minimum, with no L2 and no complex coherence path to traverse. Then absorb what's left dynamically with an out-of-order window and a decoupled prefetch unit — independent instructions keep issuing while a miss is outstanding. It demands neither high occupancy nor big tiles, so at batch=1 and small shapes it can still sit near the roofline. This is the mechanism behind Jalapeño's 700+ tokens/s/user at concurrency 1.

"Out-of-order" here most likely isn't the Tomasulo kind. The source material's own words are that each core provides data prefetch and a decoupled out-of-order unit, and that you can wait on prefetched data and lock it with semaphores. Software-visible waits and semaphores say this isn't hardware OoO that's transparent to software; it's closer to Smith's 1982 decoupled access-execute: an address-generation unit runs ahead, memory returns can complete out of order, and the compute pipeline lines up via a scoreboard plus explicit waits.

**Where the cost lands:** everything is bet on prefetch accuracy. Static reasoning gets harder — you can't look at a DMA timing table and know whether a kernel will stall. And each core carries the area and power of OoO logic and an L1, which in DSA circles has traditionally been judged not worth it. But that's also the point of what comes next: the kernels are now being written by AI.

## Why you need the compiler's view

Give up the static analyzability of a scratchpad and half of the mature NPU compiler toolchain stops working. Under the scratchpad model, tile-size search is an integer program whose constraints you can write in closed form — capacity, MAC granularity, DMA efficiency, double-buffering, four sets of constraints solved jointly by ILP or exhaustive search; instruction scheduling uses modulo scheduling to find the minimum initiation interval and can approach the hardware's concurrency limit. Both passes rest on the same premise: memory latency is a known constant.

With a cache, that constant becomes a distribution that moves with prefetch accuracy. Tile search degrades into profile-driven autotuning (the TVM auto-scheduler, Triton autotune family), and the scheduler loses its basis for static pipelining. OpenAI's hedge is to hand the whole grind to an agent: Codex, plus a harness with detailed traces, automatically searching for the best prefetch plan. That's really the same problem modulo scheduling solves, swapped for a profile-in-the-loop black-box search — the same question, just retreated from analytic solution to measured iteration.

The whole scheme is bet on prefetch accuracy, and you can't tell in advance, DMA-timing-table style, whether a given kernel will stall. In the traditional setting that alone would basically veto the design — an architecture whose performance leans this hard on prefetch timing means a human hand-tunes every shape, and the tuning doesn't generalize.

![GEMV at M=1: a DMA-fed systolic array idles between tiles; a cache-fed OoO core keeps issuing through a miss, at the cost of unpredictable stalls.](/assets/images/posts/jalapeno-built-for-batch-size-1/04-gemv-timing-dma-vs-ooo.png)

### A few system-level pieces

Beyond the microarchitecture, there are a couple of places in the system design that only pay off under a low-batch assumption. They're different in kind, but clearer side by side.

First, the interconnect. Each ASIC has 4.8 Tb/s per direction, all-to-all into six Tomahawk 6 switches, and the bandwidth allocation explicitly favors tensor parallel over expert parallel.

The chain of logic: at concurrency 1 you have no data parallelism to use, so the only way to speed up is more TP; and TP does an all-reduce for every generated token, so inter-core communication latency goes straight into TPOT. Within eight days of first silicon they scaled TP8 to TP32 and ran a rack-scale large model — that's exactly the path being validated.

![How TP, EP and DP behave in prefill (GEMM) versus decode (GEMV). In decode, TP is the only lever and DP does nothing for TPOT — hence TP high, EP low.](/assets/images/posts/jalapeno-built-for-batch-size-1/05-tp-ep-dp-matrix.png)

The scale-up network also has headroom left in it: it's about 10% of system cost, and what it buys is room for 10–20 trillion-parameter models and 2–4 million tokens of context. That second half effectively names the target workload: other than agentic, I can't think of anything that needs to run to four million tokens.

---

## AI designing the software and the hardware

The official "nine months" refers to RTL to manufacturing tape-out; Broadcom calls it possibly the fastest ASIC development cycle in high-performance advanced semiconductors. The full timeline is longer: design work started in mid-2024, and from staffing up to tape-out was about 16 months. The CoWoS tape-out completed in November 2025 — tape-out of the whole CoWoS design, note, not just the top-level die. By the time they were on stage at Hot Chips, bring-up on actual silicon had run for only three months.

![Tape-out pace against the incumbents. Rubin taped out a month earlier; Jalapeño had third-party benchmarks three months into bring-up.](/assets/images/posts/jalapeno-built-for-batch-size-1/06-tapeout-timeline-vs-rubin.png)

For reference, NVIDIA Rubin's CoWoS tape-out was October 2025 — a month *earlier* than Jalapeño.

That comparison matters because it turns "AI accelerated the design" from a marketing line into a testable claim. A team starting from zero, with a completely empty software stack, had outside parties in the lab running benchmarks three months into bring-up; a far more mature competitor, over the same window, released only early results from engineering samples in partners' hands. Where the difference comes from is what the rest of this piece takes apart.

## The hardware side

OpenAI used the XLS hardware-description toolchain, with internal models to accelerate the flow. XLS is Google's open-source high-level synthesis toolchain, Apache 2 licensed, generating synthesizable Verilog/SystemVerilog from a high-level functional description. Its front end, DSLX, is a domain-specific language that deliberately mimics Rust syntax, aimed at dataflow-style hardware description.

So picking XLS gets two things at once:

1. **A higher level of abstraction.** What the model generates is closer to intent than to cycle-by-cycle timing. The probability of a correct generation goes up; the cost of localizing an error goes down.
2. **A free ride on training data.** DSLX looks like Rust, and the volume of public Rust code dwarfs DSLX itself.

## The part I'm unsure about: how do you write "out-of-order" hardware in XLS?

What's hard about a real OoO core is register renaming, the ROB, the single-cycle wakeup-select loop in the issue queue, speculative load-store disambiguation — structures where **the timing is the semantics**, and HLS can't help.

**My guess at where XLS's boundary lies:**

Writable in DSLX: decode, address generation, MSHR allocation and matching logic, cache tag pipelines, arbitration, MAC datapaths, network interfaces. These are all feed-forward dataflow; give it a clock target and XLS's scheduler decides how many register stages to insert.

Not writable: CAM match arrays, multi-ported register files, bypass networks, SRAM macros, clock trees. Those come from a memory compiler, or hand-written SystemVerilog, or straight from Broadcom's hard IP. HLS emits Verilog anyway; stitching it together with hand-written blocks is routine.

## Process: the spec converges inside the loop, not before it

OpenAI's statement of methodology at Hot Chips was blunt: the spec isn't fully known up front; it converges through the design loop. What they emphasized was the measure–verify–learn–modify–repeat cycle, with the goal of shortening the feedback loop between workload simulation, RTL, QoR analysis, DV, and physical design.

That's a genuine methodological shift, not just tooling efficiency. The reason the traditional ASIC flow freezes the spec early is that the cost of late changes grows exponentially — every RTL change means re-running synthesis, redoing timing closure, re-running regressions. When each turn of that loop compresses from person-weeks to machine-hours, it really does upend the conventional design flow.

![The design loop, with a feedback path measured in machine-hours. Compress one turn that far and the case for freezing the spec early disappears.](/assets/images/posts/jalapeno-built-for-batch-size-1/07-design-loop.png)

### Gluon and Linear Layouts

Jalapeño's kernels are written in Gluon. Gluon is built on Triton and keeps Triton's SPMD programming model, but exposes the lower-level abstractions — on NVIDIA GPUs, for instance, it offers APIs that map directly to PTX instructions, covering MMA, TMA, mbarrier, and so on.

Gluon's most distinctive abstraction is the **layout**: a mapping between hardware resources (say, register *k* of warp *j*) and tensor elements (row *r*, column *c*). And Gluon's layout abstraction is built on Linear Layouts — a layout algebra OpenAI invented, which formalizes mathematically what a layout *is* and provides tools for computing on layouts.

Two capabilities this algebra gives you are the key to the whole thing: **provably correct layout conversions**, and **optimal memory swizzling**.

Why does that matter? Because when you hand code generation to a model, the biggest risk was never "it can't generate anything." It's "it generated something wrong, and not obviously wrong." Linear Layouts draws an automatically checkable correctness boundary exactly where subtle errors are most likely — how data sits on the hardware, how it changes layout. With that boundary in place, the model's search becomes finding the performance optimum inside a safe space, rather than gambling in an unconstrained one.

The supporting mechanisms: each Gluon program maps to a persistent thread, which means the chip suits a persistent-kernel style — tiles assigned by the programmer (or rather, by the model generating the program) instead of a hardware scheduler; a TensorInfo abstraction that explicitly encodes layout; and per-core data prefetch with a decoupled out-of-order unit, where you can wait on prefetched data and lock it with semaphores.

![The same three moves on both sides: raise the abstraction, draw a checkable correctness boundary, shorten the feedback loop.](/assets/images/posts/jalapeno-built-for-batch-size-1/08-hardware-software-playbook.png)

### Pre-silicon exploration: how the Triton layer and the XLS layer could talk

I have no public material for this section; everything here is my own guess at the AI flow, extrapolated from the current one. How an architecture team — the advance scouts for a product still in research — can use AI to speed up validation and exploration is a direction worth thinking about in its own right.

**Junction one: the ISA as single source of truth.** DSLX is purely functional; write down an instruction's semantics and you have a function — and that same function can generate the decoder RTL, the compiler back end's instruction description, the assembler, and the golden reference model. This is what Arm does with ASL and the RISC-V ecosystem does with Sail. For custom silicon the ISA isn't a given, so the significance of this layer is that you change an instruction's semantics once and the change propagates to the silicon and the compiler together, instead of each side changing separately and discovering the mismatch at bring-up.

**Junction two: one IR, two outputs.** XLS's IR lowers to Verilog, but it also has an interpreter and a JIT. That means a fast model, same-sourced with the RTL, exists before tape-out. Gluon kernels can run on hardware that hasn't been built yet and produce traces — not Verilator's ten-thousand-times-slower simulation, but a speed that fits inside a search loop. This is the **only** way to pull Codex's profile-in-the-loop forward to before first silicon. Otherwise that black-box search can't start until there's silicon, and three months of bring-up wouldn't get you to a third-party benchmark.

**Junction three: the two searches were always coupled.** The hardware side searches discrete parameters (window depth, MSHR count, cache geometry, TP link width), scored by PPA plus workload simulation; the software side searches schedules (tile size, prefetch distance, semaphore placement), scored by measured cycles. The best kernel schedule depends on how deep the window is, and how deep the window should be depends on what schedule the kernel can produce. Traditionally you break the cycle by freezing the hardware, at the cost of software only ever reaching a local optimum on given hardware. Only when both sides can be machine-searched *and* share the same simulator can you talk about joint optimization. I think that's what "the spec converges in the design loop" actually means — not "the requirements weren't thought through."

**Junction four: the two correctness boundaries can be joined.** What Linear Layouts formalizes is the tensor-coordinate ↔ hardware-resource mapping, and the "hardware resource" it refers to (register *k* of warp *j*) has a precise definition on the XLS side. If the resource models on both sides come from the same microarchitecture description, then the provable correctness of a layout conversion is **proved against real silicon**, not against a model in someone's head. This is the possibility I find most interesting — and the one I'm least sure of.

**The most concrete contract is in the waits and semaphores.** Gluon issues the waits; DSLX implements the scoreboard they gate. Adding a new form of wait (say, "wait until N outstanding") is a change to both sides at once — and only affordable when both sides can be regenerated cheaply. This is the most tangible interface of hardware–software co-design.

![A speculative picture of how the XLS and Triton layers could connect. The dashed loop back to the ISA is my addition; the wait/semaphore contract is where the two sides actually meet.](/assets/images/posts/jalapeno-built-for-batch-size-1/09-four-junctions.png)

One thing in the diagram is mine, not from the source: the dashed line in the middle that loops back up to the ISA description. Parameter search reaches a point where tuning the window depth alone isn't enough — you need to add an instruction to the ISA or change the semantics of a wait — and the loop wraps back to the top box. That layer iterates far less often than the two loops below it, but whether it exists decides whether "the spec converges in the loop" is a true statement or a nice-sounding one.

Two caveats. First, XLS's scheduler trips on complex proc networks — a known case is a systolic array's FIFO configuration getting scheduled into a cyclic dependency that won't lower to Verilog. So "higher abstraction" isn't free: it moves some of the debugging from the RTL layer to the tool layer, where documentation and diagnosability are usually worse. Second, of the four junctions above, only the second — a simulator from the same-sourced IR — has a concrete tool capability behind it. The other three are architecturally sound but unevidenced inference. Weight them accordingly; they are not all the same strength.
