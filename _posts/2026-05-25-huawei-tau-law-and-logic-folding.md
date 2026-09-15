---
layout: post
title: "Huawei's \"Tau's Law\" and Logic Folding: What's Actually New"
date: 2026-05-25 09:00:00
categories: [semiconductors, packaging]
excerpt: "Read past the \"Chinese Moore's Law\" framing. The four-level co-optimization is Intel's post-2018 playbook run by choice rather than necessity; the engineering core is sub-2 μm hybrid bonding; and the counterintuitive part is that a phone gets it before the cloud does."
---

*Read past the "Chinese Moore's Law" framing. The four-level co-optimization is Intel's post-2018 playbook run by choice rather than necessity; the engineering core is sub-2 μm hybrid bonding; and the counterintuitive part is that a phone gets it before the cloud does.*

Seeing this news, an architecture person with a microelectronics background can only rejoice: the future really is the era of co-innovation, and the pace at which large models replace us will be a little slower, haha.

*This is a translation of a Zhihu answer originally published on May 25, 2026.*

Public reaction has mostly stuck to a sloganized reading of "China's Moore's Law." To see what this announcement really means, you first have to make a finer industrial analogy — and my hot take is that what Tau's Law really maps onto is the "three-legged" strategy Intel worked out after 2018, when process dragged it down.

Intel at its peak ran on the Tick-Tock cadence: a new process every two years, with architecture iterating on the new process. That model failed completely after 2015 — 10 nm slipped, 7 nm slipped — and Intel had to rebuild its whole technology strategy. Its response can be split into three legs:

Leg one, heterogeneity on the architecture side — the P-core + E-core hybrid (rolled out to mainstream desktop from Alder Lake), Thread Director scheduling, mixing general-purpose compute with specialized accelerators.

Leg two, non-geometric innovation on the process side — PowerVia backside power delivery (introduced at 18A), RibbonFET gate-all-around transistors. These no longer rely on geometric shrink; they change the device structure itself.

![Intel's three legs](/assets/images/posts/huawei-tau-law-and-logic-folding/75069744a91ec07f9866285ca51ad0e0.jpg)

Leg three, and the most critical, going vertical on the packaging side — EMIB (2.5D bridging) → Foveros (3D logic stacking) → Foveros Direct (bumpless direct connection based on hybrid bonding).

Look back at Tau's Law's four-level co-optimization system — device, circuit, chip, system — and it closely resembles Intel's combination. There isn't much novelty in this announcement; it's more that China is defining its own industrial methodology for the first time. Intel did it out of necessity after losing its process lead; Huawei is doing it as a deliberate choice, never having had a process lead. The former is a patch; the latter is a path.

![The four-level formulation of Tau's Law](/assets/images/posts/huawei-tau-law-and-logic-folding/tau-formula.png)

### 1. The RC product

To understand Tau's Law's technical core, go back to the most basic fact in circuits: τ = RC.

The time constant τ, the product of resistance R and capacitance C, sets the propagation delay of a signal on a conductor. The essential bottleneck of chip performance was never how fast a transistor switches; it's how slowly a signal travels on the interconnect. Below 5 nm, interconnect RC delay's share of the critical path already exceeds 50%. In other words, geometric shrink has physically stopped solving the performance problem — it's making it worse: thinner wires mean more resistance, denser wires more capacitance.

![RC discharge: the time constant is where the delay lives](/assets/images/posts/huawei-tau-law-and-logic-folding/rc-discharge.png)

Tau's Law's proposal to "replace geometric scaling with time scaling" is, at bottom, an acknowledgment of this physical reality, shifting the optimization target from transistor size to the time constant τ. It's a fundamental change of approach: no longer chasing smaller, but chasing faster signal propagation.

The concrete path is four interlocking gears: device, circuit, architecture, system. The effect of logic folding at the circuit level, for example, is constrained by the RC characteristics of the transistors at the device level, and in turn affects instruction scheduling at the chip level and thermal-budget allocation at the system level.

This cross-level coupling isn't new, but the SRAM part interests me. Large SRAM's importance to AI inference — the largest compute market right now — may be logic folding's real industrial landing point. LLM inference is bottlenecked by memory bandwidth, not compute; most of execution time is spent waiting for data to move from HBM to the compute units. Total bandwidth from the several HBM stacks on a high-end AI chip is typically 3–8 TB/s; on-chip SRAM can reach tens of TB/s — an order of magnitude. The bigger the weights, the longer the context, the larger the batch, the narrower the HBM gate. Multi-batch long context is especially hard, because the KV cache grows linearly with batch and sequence length: a 70B model at 32K context and batch 8 has a KV cache near 80 GB, far beyond any on-chip SRAM, so it has to go through HBM. In that situation, the larger the on-chip SRAM, the more times each weight and activation moved up from HBM can be reused, the higher the memory hit rate in attention, and the faster the overall throughput. Groq's LPU running inference on 230 MB of on-chip SRAM with no HBM at all is the extreme version of this idea; Cerebras's WSE with 44 GB of SRAM on one wafer is the same logic.

Logic folding's double boost to large SRAM's frequency and density lands exactly on the core bottleneck of this route. Fold a long-line large SRAM array and the bitlines and wordlines shorten by about half, improving both the bandwidth ceiling and capacity — a physical gain that V-Cache-style "stack more on top" can't buy. If the Ascend series can use full logic folding after 2027, the biggest beneficiary won't be compute; it will be the effective memory-bandwidth gain from doubling on-chip SRAM.

### 2. Hybrid bonding

If the four-level system is Tau's Law's methodological skeleton, logic folding is its engineering core. First, the gap between conventional 3D stacking with TSVs and hybrid bonding.

The technical parameters, compared:

| Dimension | TSV (with μbump) | Hybrid bonding (Cu–Cu direct) |
| --- | --- | --- |
| Interconnect pitch | 40–55 μm (μbump level); TSV diameter 10–100 μm | TSMC SoIC in production at 6 μm; Intel Foveros Direct targeting sub-5 μm in H2 2026; Sony at 1 μm in research |
| I/O density | hundreds to thousands per mm² | tens of thousands to hundreds of thousands per mm² |
| Parasitic RC | significant bump inductance + TSV capacitance | close to on-die wiring |
| Interconnect delay | hundreds of ps | tens of ps, close to on-die |
| Thermal resistance | air gaps between μbumps, poor conduction | all-metal / dielectric interface, far better heat path |
| Manufacturing bar | mature; widely used in HBM and CIS | needs CMP-grade surface roughness, sub-μm alignment, stringent cleanliness |
| Yield risk | single-point failures relatively cheap | two wafers' yields multiply |
| Typical uses | HBM stacks, CIS, 2.5D interposers | TSMC SoIC, Intel Foveros Direct, AMD V-Cache, HBM4 onward |

**Density:** a 7 nm logic chip has about 90M transistors per mm², while conventional μbumps provide only a few hundred I/Os per mm². Connect two logic layers with TSVs and the I/O can't remotely keep up with inter-layer data exchange; you can't distribute one logic chip's functional blocks across two layers. That's why TSVs suit only memory-on-logic (HBM, V-Cache), where inter-layer communication is localized. (Strictly, V-Cache is logic-on-logic at the process level, but Huawei's version, building on that concept, leans more toward compute-on-compute.)

**Delay and frequency:** Tau's Law is about lowering the RC time constant. The series parasitic RC of TSV + μbump is often an order of magnitude or more above hybrid bonding's. If the folded critical path had to cross a TSV, the delay saved by "shorter wiring from folding" might not even fill the hole dug by TSV + μbump delay. This is the most contested point about the technology: 3D stacking normally lowers frequency, but here it rises, relying on system-level delay reduction and the introduction of hybrid bonding.

**Power:** the pJ/bit of SerDes driving μbump + TSV is far above hybrid bonding's (typically 5–10×). In a phone SoC, with its extremely tight power constraints, a TSV solution could never "fully adopt logic folding." I read a paper from Prof. Sun's group at Peking University on hybrid bonding at the edge — it won ISCA 2025 best paper.

There's one more account that's the most worth expanding in Huawei's disclosure: the BEOL matching account. He Tingbo laid out an engineering principle in the talk: the ratio of hybrid-bonding pitch to top-metal pitch should be kept under 3, the closer to 1 the better. Current top-metal pitch is about 720 nm, so hybrid-bonding pitch has to be kept under 2 μm. That principle explains something most people miss: why TSMC SoIC's 6 μm pitch is enough for V-Cache but not for compute-on-compute logic folding. V-Cache's top layer is SRAM, which doesn't need such dense metal; 6 μm to 720 nm is about 8:1, barely acceptable. But in compute-on-compute folding both dies have 720 nm-class dense metal, the ratio has to drop under 3, so hybrid-bonding pitch has to reach under 2 μm. That's also why Huawei's set of numbers — 2 μm / 1.5 μm / 6 μm / 0.5 μm — is self-consistent in engineering terms, not a few numbers picked at random.

### 3. The domestic supply chain

We lag in process, but in advanced packaging we're actually strong. On hybrid bonding, China isn't a follower; it's one of the early definers. As with DeepSeek before it, this announcement rests on China's complete supply chain and breakthroughs in particular segments.

The representative player is YMTC. Per the French patent-analysis firm KnowMade, YMTC published 119 hybrid-bonding-related patents between 2017 and January 2024; by comparison Samsung Electronics, despite filing since 2015, had only 83 by the end of 2023, and SK hynix only 11. This is a number worth repeating: it means the patent moat in hybrid bonding currently belongs to a Chinese company. Samsung is reported to be licensing hybrid-bonding patents from YMTC for its next-generation NAND, and SK hynix is in follow-on talks. A Chinese vendor reverse-licensing to the Korean giants is extremely rare in semiconductors. (Maybe Chinese vendors really can overtake on the memory curve and blow up US stocks, hhh /dog-head.)

Technically, YMTC's Xtacking is at its fourth generation. Its fifth-generation NAND, shipping from early 2025, uses Xtacking 4.0 at 294 layers. In 2025 YMTC also worked with CXMT to push hybrid bonding into HBM — meaning the technology is extending from NAND toward DRAM/HBM, on pace with the world. This is my own field; I've read many of Prof. Sun Guangyu's papers at PKU, and much of the unpublished part looks like collaboration with these commercial companies.

Now let me play contrarian on some possible weaknesses.

The first gap is the span of application. Xtacking is W2W hybrid bonding for memory, with bond pitch at several microns to ten microns, mainly carrying the connection between the NAND cell array and the peripheral CMOS; I/Os per mm² are far below what logic folding needs. The 2 μm-class bond pitch Huawei disclosed is for logic-on-logic, with requirements far stricter than NAND's hybrid bonding — harder on thermal stress, parasitics, and multiplied yield all at once. Going from mature NAND experience to logic chips isn't a simple process transfer; it's nearly re-doing the process-window validation from scratch.

The second gap is equipment. Production-grade hybrid bonders worldwide come almost entirely from BESI and ASMPT. China has made progress in sub-steps — precision alignment, CMP, cleaning — with players like Huazhuo Jingke and ACM Research, but there's no domestic equivalent of a complete production-grade W2W hybrid bonder yet. Fortunately both equipment vendors can still supply Chinese customers; BESI and ASMPT aren't on the strictest export-control lists.

The third gap is verification. Huawei's sub-2 μm hybrid-bonding data came from a talk, not a peer-reviewed paper or a third-party teardown. After Kirin 2026 launches in the fall, the die-shot analyses from TechInsights and SystemPlus will be the real reckoning. Until then, every judgment should carry the qualifier "Huawei claims."

If the data holds, domestic hybrid bonding is one of the few areas where China has world-class capability on all three dimensions — patents, specs, application experience — lacking only domestic replacement of the equipment. If the data is doubtful, the Kirin 2026 measurements will give the final answer. Either way, hybrid bonding is currently the technical lever Chinese semiconductors are least choked on, and that's the key support for judging Tau's Law's feasibility.

### 4. Heat

Another hot take: when Tau's Law lands in actual products, there's a counterintuitive judgment to make explicit. Logic folding will land more smoothly in edge phones than in cloud AI accelerators.

That runs against most people's intuition. The usual assumption is that the cloud has better cooling and should be the debut venue for advanced technology. Engineering reality is the opposite.

The cloud case is Ascend 910, 920, and future higher parts. 300–700 W per chip, liquid cooling available, high board-level freedom in thermal design — the cooling conditions look superior. The problem is that once hybrid bonding stacks two logic layers, local power density doubles outright, from a typical 1 W/mm² to over 2 W/mm². The top die's heat path has to cross the bottom die or a TSV network, and thermal resistance worsens markedly. Solving that needs backside liquid cooling, like the micro-channel cooling TSMC is researching, or embedded micro-channel packaging, or folding only low-activity dies — on-chip SRAM, I/O controllers. All of these are still at the research or early-engineering stage, with a production bar higher than hybrid bonding itself. The cloud is actually not the best debut venue for logic folding.

The edge case is this generation's Kirin 2026 phone chip. Whole-device TDP of 4–8 W sustained, passive cooling, an extremely tight thermal budget. On the surface, harder. But a phone chip's bottleneck is performance per watt, not absolute power. Tau's Law's core value is lowering τ without shrinking dimensions, which in engineering terms means reaching the same performance at a lower clock with a shorter critical path. That's a direct win for power.

The process gap is also biggest in phone SoCs. SMIC's N+2 versus TSMC N3 is about a 1.5-generation gap; that's where Huawei most needs folding to make up for process. The thermal strategy is asymmetric folding: stack low-transient-power-density blocks — SRAM, cache, ISP, modem — above the CPU or GPU, rather than compute-on-compute. This cool-block-over-hot-block layout captures hybrid bonding's density gain while keeping the worsening of heat density within acceptable range.

AMD has been validating this strategy on V-Cache for about four years. AMD places the SRAM die precisely centered over the CCD's L3 region, away from the heat-generating CPU cores on either side. Kirin 2026 most likely takes the same road, except that what gets stacked on top isn't limited to SRAM. If Huawei's sub-2 μm bond pitch really can be produced stably, its flexibility in stacking choices will be an order of magnitude greater than V-Cache's, because a finer pitch means more kinds of functional blocks can be folded.

So my guess is that choosing Kirin 2026 to debut full logic folding is the most rational choice in engineering economics: the largest process gap, the most painful performance bottleneck, a relatively controllable heat-density problem, and enough volume to amortize yield risk. Ascend, by contrast, has to wait until the supporting cooling matures in 2027–2028 to use full folding. He Tingbo says the next decade heads toward full folding; in the first few years it's most likely phones first, cloud following.

### 5. From technology to people

The sections above analyze how the technology is done; this one is about how people change. What does Tau's Law's four-level co-design system mean for people working in chip design? My judgment is that it will accelerate the differentiation of chip designers, and the dividing line of that differentiation happens to be the threshold of what large models can replace.

Over the past three years, large models have penetrated chip design faster than most expected. NVIDIA's ChipNeMo, Cadence Cerebrus AI Studio, Synopsys DSO.ai — these tools have already largely automated RTL generation, EDA scripting, bug summarization, testbench generation, and floorplan exploration. Per Cadence's public data, Cerebrus has already shrunk a chip block's area by 5% and cut power by over 6% on one SoC. The newly launched Cerebrus AI Studio claims to accelerate SoC delivery 5–10×. Academia is more aggressive: ChipSeek-R1, published in 2025, is already trying to train an LLM specialized for RTL generation with hierarchical reinforcement learning, and claims to beat human designers on some benchmarks.

These AI tools share one trait: they're good at optimization problems with clear boundaries and a single objective. RTL generation has a clear spec boundary and correctness verifiable by testbench. Floorplan optimization has a quantifiable PPA objective. Timing closure has explicit constraints and a structured search space. Verification has measurable coverage. These tasks happen to be where the chip-design industry's job growth of the past decade went; lots of young engineers do exactly these.

The capabilities Tau's Law demands are exactly the reverse: cross-level, fuzzy-objective, far-reaching judgment. Should this block be folded? Weigh the delay gain, the heat-density penalty, the yield cost, the EDA toolchain's maturity. What does the physical critical path look like after folding? That requires anticipating back-end consequences at the architecture level. How does the Lingqu bus protocol cooperate with this chip's cache coherence? That requires understanding semantics both on-chip and across the system. When the thermal budget is broken, do you back off the architecture or change the packaging? That's a joint business, process, and design decision. The chip engineer's skill isn't only the care to beat LLM hallucination; it's carrying responsibility for a failed tape-out (kidding — not kidding).

No large model can currently make this class of decision reliably. The reason isn't model size; it's that these decisions require cross-domain trade-offs without an explicit objective function. That's the steepest part of the LLM capability curve right now. You can give it a spec and have it write code, but you can't give it a multi-objective "I want this and that and also that" problem and have it make the architectural call.

Tau's Law isn't only an industrial strategy for Chinese chips; it's also a capability strategy for individual chip designers. The full-stack awareness it demands overlaps heavily with the capability dimensions needed to resist replacement by LLMs. That's no coincidence; it reflects the same deep industrial logic. When the Moore's Law dividend fades and geometric scaling gives way to system co-design, the people who can master system co-design become the scarcest part of the system. Tau's Law is both the Chinese semiconductor industry's methodology against process blockade and the chip designer's methodology against AI replacement.

*Lastly — though the acceptance rate is a bit high, ISCAS really is a good conference; people share real work there /dog-head.*
