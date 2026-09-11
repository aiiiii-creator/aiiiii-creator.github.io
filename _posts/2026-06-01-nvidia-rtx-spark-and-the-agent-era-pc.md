---
layout: post
title: "NVIDIA's RTX Spark and the Agent-Era PC"
date: 2026-06-01
categories: [hardware, pc]
excerpt: "Why NVIDIA is entering PCs when every wafer is worth more in a data center, why Arm, and seven metrics for judging an AI PC once the workload is agents running for hours rather than one-shot chat."
---

*Why NVIDIA is entering PCs when every wafer is worth more in a data center, why Arm, and seven metrics for judging an AI PC once the workload is agents running for hours rather than one-shot chat.*

At Computex 2026, a day after Microsoft and Arm's "A new era of PC" teaser, NVIDIA's RTX Spark made its official debut. It integrates a Blackwell GPU, a 20-core Grace CPU, and 128 GB of unified memory, delivers 1 PetaFLOP of AI compute, and runs Windows on Arm.

*This is a translation of a Zhihu answer originally published on June 1, 2026.*

### 1. Crushing the consumer business from above?

Since 2025, capacity for both compute and memory chips has tilted toward the data center. Samsung, SK hynix, and Micron have shifted more advanced-node capacity to high-bandwidth memory and high-end DDR5 and actively cut consumer lines. The reason: one AI server needs 8× the DRAM and 3× the NAND of an ordinary server, and data-center customers have long order cycles and high margins. The result is tightening consumer memory supply and rising prices; the consumer segment is under 3% of the industry mix and ranks behind the data center in capacity allocation.

![](/assets/images/posts/nvidia-rtx-spark-and-the-agent-era-pc/a180e8418d6d96287d63578b4cec6b46.jpg)

Against that backdrop, capacity tightness itself is not a motive for NVIDIA to enter PCs. When capacity is constrained, every advanced-node wafer NVIDIA spends on PC chips is one fewer set of data-center product, and a data-center system's unit gross profit is several times a consumer chip's. By profit-maximization logic, wafers should go to the data center first. So the capacity shortage suppresses PC entry rather than driving it.

My guess is that NVIDIA's motive for entering PCs now comes from two places. First, agents. Coding is one of the most mature deployment scenarios for large models; if part of the inference load migrates from the cloud back to the device, the device becomes a link NVIDIA needs to cover — consistent with its overall strategy (it's also laying out robotics and autonomous-driving chips). Second, the software ecosystem. NVIDIA's core moat is CUDA, and its direction is to make CUDA as universal as possible; extending CUDA from the cloud to edge devices grows the developer base. RTX Spark's AI compute comes from the GPU, not a separate NPU, so the on-device workload stays inside the CUDA system — something Qualcomm's and Apple's NPU routes don't have.

![](/assets/images/posts/nvidia-rtx-spark-and-the-agent-era-pc/4c482a4e4da83715b05266a92d00b22b.jpg)

Back to the title question: NVIDIA's entry into the consumer side doesn't ride on surplus capacity steamrolling the market; the opportunity cost above is a headwind. But in the capacity-acquisition link, NVIDIA does have an edge over other consumer-electronics vendors. Memory price rises put memory-dependent consumer devices broadly under pressure in the first half of 2026, while NVIDIA, as one of TSMC's main customers, has priority on advanced-node and packaging capacity, and by partnering with MediaTek to share CPU design and part of the supply chain, it can secure a 3 nm allocation. As for the mode of entry, RTX Spark is positioned for the high-end productivity and creator market, against Apple's M5 Pro and M5 Max, with first products in the fall and MediaTek participating. That combination is the strategy under constrained capacity: enter at the high-value high end rather than chase unit volume.

### 2. The AI PC and the Arm ISA

For forty years, Windows PC processors have been essentially x86, an ISA held by Intel and AMD through cross-licensing, hard for third parties to enter. Arm is another ISA, on a broadly licensed model; Apple, Qualcomm, NVIDIA, MediaTek, and others can all design their own processors on Arm. Three vendors turning to Arm for this generation of PC products changes the long-standing x86 makeup of Windows PCs.

![](/assets/images/posts/nvidia-rtx-spark-and-the-agent-era-pc/0563223c3bff6b6403e36c356b48ca10.jpg)

Arm's low-power character comes from the devices it originally served. Arm is a RISC architecture whose energy efficiency originally served battery-powered mobile and embedded devices, where it dominates; its entry into the data center is a later extension. That is, low power is a trait Arm formed on personal and portable devices, not something exclusive to the data center. Laptops likewise need to control power beyond performance to sustain battery life, which matches Arm to the personal PC.

NVIDIA's entry doesn't necessarily squeeze Qualcomm and Apple, for two reasons. First, different technical positioning. Apple's M5 runs closed macOS and has the highest memory bandwidth of the three; Qualcomm's Snapdragon X2 targets mainstream Windows thin-and-lights, with the most complete published CPU single-core and efficiency data; NVIDIA's RTX Spark targets high-end Windows creation and AI development, with AI compute from the GPU inside the CUDA ecosystem. The three don't fully overlap in users. Second, market expansion. Arm's long-standing obstacle in Windows has been x86 software compatibility; more vendors entering Arm PCs accelerates software and driver adaptation to Arm, lowering that obstacle — a process that benefits every vendor in the Arm camp. So at this stage, the three entering together is more about jointly growing Arm's share of personal PCs than redistributing a fixed market.

### 3. Is the AI PC still just a concept?

In my view, the previous AI PC was largely a concept, while the introduction of agents brings real productivity improvement. At the same time, current AI-application penetration isn't high, and there's further room.

![](/assets/images/posts/nvidia-rtx-spark-and-the-agent-era-pc/9f5122cf7d3e552e728f67ad54db30a5.jpg)

The main factor changing PC usage this year is the spread of AI agents. Last year's AI PC was mostly single-turn Q&A: the user types a question, the model returns an answer, then waits for the next input. In that mode, compute occupancy is pulse-like — the model saturates the hardware during inference and hits peak during benchmarks, while between inputs the hardware sits mostly idle. Whole-machine load is intermittent, peak performance determines single-response speed, and average utilization isn't high.

This year's agents change that pattern. An agent is no longer a passive chat window waiting for input; it's a program that advances a task on its own. It runs in the background for a long time, decides its next action by the goal, calls tools to execute operations, reads documents and web pages, generates and runs code, and switches among multiple tasks. One agent completing one task often takes many rounds of reasoning, many tool calls, and repeated processing of intermediate results, lasting minutes to hours rather than one exchange. With several agents running at once, the hardware has to hold multiple model states and contexts continuously. The AI PC's operating mode thus shifts from intermittent use to long, continuous, multi-task operation.

That shift changes the criteria for hardware. In intermittent use, a machine is judged mainly on single-inference peak — peak compute and peak clock. In agent mode, the load becomes long, multi-task, continuous operation with frequent handoffs between CPU and accelerator, and the factors deciding the experience change with it. Whether the model and several agents' states fit in memory at once depends on memory capacity. How fast an agent keeps generating depends on decode-stage memory bandwidth. How fast long context and several parallel tasks are processed depends on peak compute. The overhead of frequent CPU–accelerator switching depends on the interconnect between them. The speed of serial logic like tool calls and scheduling depends on CPU single-core. And whether the machine runs at full load for hours without throttling depends on efficiency and cooling. These metrics together decide agent-mode performance; a single peak-compute number no longer represents a machine's overall capability.

### 4. The metrics

Before comparing the three chips, set the dimensions. In my view, the agent workload has several characteristics: the model resident in memory (weights), context accumulating across turns (KV cache), multiple tasks in parallel (multi-agent), frequent CPU–accelerator handoffs (the CPU-bound links in an agent), and long continuous operation. Mapped to hardware, these split into seven metrics; each is defined below with the three products compared.

![Seven metrics, three chips](/assets/images/posts/nvidia-rtx-spark-and-the-agent-era-pc/660749f17cd2f5ec9f0f2eec2aaa5f50.jpg)

#### Memory capacity

An agent's memory footprint has three parts: the model weights, the KV cache for the context, and each agent's own state. Weights stay resident after loading. The KV cache grows linearly with context length; a million-token context corresponds to tens of GB of KV cache. With several agents in parallel, each maintains its own context. Stacked together, these push memory use an order of magnitude above a single conversation. Memory capacity decides how large a model and how many agents a machine can hold at once. When capacity is short, the model can't load, or the context is truncated and the agent loses its earlier state.

The three flagships have the same unified memory capacity, 128 GB. NVIDIA RTX Spark has 128 GB of unified memory shared by CPU and GPU. Qualcomm Snapdragon X2 can be configured at 128 GB and above — on the motherboard for the X2 Elite, in the SoC package for the X2 Elite Extreme. Apple M5 Max is 128 GB; M5 Pro is 64 GB. On this metric the three flagships are level; it isn't a differentiator.

#### Memory bandwidth

In typical personal scenarios, a large model's autoregressive decode at batch 1 is memory-bound. Each generated token requires reading the entire model's weights from memory once; arithmetic intensity is low and the bottleneck is bandwidth, not compute. Agent output is mostly long text — code, plans, multi-step reasoning — and each step generates many tokens. So memory bandwidth decides how fast an agent generates, which is how fast it executes tasks. To be clear, I think future multi-agent and some personal applications (email summarization, say) will lean more compute-bound, but even so, memory bandwidth remains the key metric in decode.

Apple M5 Max is 614 GB/s; Qualcomm Snapdragon X2 Elite Extreme is 228 GB/s; the standard X2 Elite with a 128-bit bus is about 152 GB/s; NVIDIA RTX Spark hasn't been published. M5 Max's bandwidth is about 2.7× the Snapdragon X2 Elite Extreme's. As above, this metric decides batch-1 decode speed — local LLM chat generation speed. Of the published numbers Apple is highest, Qualcomm second, and NVIDIA's is missing, leaving its performance in this most common scenario unjudgeable — the biggest unknown among the three.

#### Peak compute

Compute is decisive in two kinds of load. First, prefill — processing the input context. Every agent turn reprocesses the accumulated history, tool returns, and retrieved documents; this stage processes all input tokens in parallel, with high arithmetic intensity — compute-bound. The longer the context and the more frequent the tool calls, the larger prefill's share. Second, acceleration techniques like speculative decoding — a small model predicts, the large model verifies several tokens in one forward pass, raising the effective batch and pulling decode from memory-bound back toward compute-bound. Multiple agents in parallel likewise raise the effective batch. All three cases give compute more weight than in single-conversation scenarios.

NVIDIA RTX Spark is 1 PetaFLOP, from a Blackwell GPU with 6,144 CUDA cores. Qualcomm Snapdragon X2 is 80 TOPS from the Hexagon NPU, at INT8. Apple M5 Max has no published absolute number, only that peak GPU compute is over 4× the previous-generation M4, spread across neural accelerators embedded in the 40-core GPU and a 16-core Neural Engine. The three source compute differently: NVIDIA from the GPU, Qualcomm from a separate NPU, Apple from GPU-embedded units plus an NPU. Among directly comparable numbers, NVIDIA's 1 PetaFLOP exceeds Qualcomm's 80 TOPS; Apple can't be placed without an absolute value. With compute from the GPU, NVIDIA's same hardware also serves 3D rendering, video, and gaming — which the other two's NPU routes don't.

#### CPU–accelerator interconnect

An agent is a mixed load alternating between CPU and accelerator. The CPU handles tool calls, parsing returns, scheduling logic, and state management; the accelerator handles inference. The two switch back and forth many times per agent loop, and the interconnect's bandwidth and access model decide the overhead per switch. The higher the switching frequency, the larger the interconnect's share of total time.

NVIDIA RTX Spark connects its CPU die and GPU die over NVLink-C2C with cache-coherent access — the only cross-die coherent interconnect of the three. Per public material on NVIDIA's data-center products (Grace Hopper, Grace Blackwell), NVLink-C2C is on the order of 900 GB/s; the consumer RTX Spark's number isn't published. Qualcomm Snapdragon X2 is a single SoC where CPU, GPU, and NPU share one memory controller; there's no cross-die movement, and the handoff bottleneck lands directly on the 228 GB/s memory bandwidth. Apple M5 Max uses a fused architecture of two interconnected dies, but the CPU, Neural Engine, and unified memory controller are on the same die, so CPU-to-NPU stays on-die. Of the three, NVIDIA achieves coherent CPU–GPU access over a dedicated die-to-die link, while Qualcomm and Apple keep CPU and accelerator on a shared or same-die path. This interconnect matters in mixed loads; in pure decode it doesn't affect the bottleneck.

#### CPU single-core

The logic connecting an agent's steps runs on the CPU: parsing tool calls from model output, executing tools, handling data formats, orchestrating communication among agents. These are mostly serial and latency-sensitive, depending on single-core performance rather than multi-core throughput. When an agent stutters, the cause may not be slow inference but a slow tool-execution and scheduling chain.

Qualcomm Snapdragon X2 uses the third-generation in-house Oryon architecture; the X2 Elite Extreme has up to 18 cores, up to 2 of them at 5 GHz, with single-core performance up to 31% over the previous generation. Apple M5 Max has 18 cores, performance cores at 4.3 GHz, in-house cores, multithreaded performance up to 15% over M4 Max, with M5 Pro and M5 Max sharing the same CPU die. NVIDIA RTX Spark has a 20-core Grace CPU co-designed with MediaTek; single-core architecture, clock, and benchmarks are all unpublished. Given that NVIDIA's data-center Grace uses stock Arm Neoverse cores, one can infer its single-core leans toward multi-core throughput and efficiency and may sit lowest of the three, but that inference needs measurement. Among published data, Qualcomm's and Apple's fully in-house cores are in the first tier, corresponding to the speed of serial tasks like tool calls and scheduling.

#### Efficiency

In last year's chat scenario, compute occupancy was pulse-like and efficiency mattered little. Agents run in the background for hours, and occupancy goes from pulse to continuous. Under sustained load, performance per watt decides the energy consumed for the same task, and whether the machine can keep working for hours without throttling. That's a new constraint the agent scenario adds over single-turn chat.

Qualcomm Snapdragon X2 Elite Extreme is up to 75% faster than Intel's and AMD's flagship mobile chips at the same power, and uses 43% less power at the same performance, with a TDP of about 50 W — the most completely disclosed efficiency data of the three. Apple hasn't published performance-per-watt for the M5 Max; its efficiency shows more in sustained performance under load, covered under cooling below. NVIDIA said at the launch that the CPU mustn't take the power the GPU needs to generate tokens, meaning the CPU's power is capped to protect the GPU's budget, but no per-watt figure was published. This metric corresponds to energy use during long background agent runs.

#### Sustained cooling

Beyond efficiency, cooling is an engineering capability outside the chip. The same chip in different chassis can perform differently under long full load. When an agent runs continuously in the background, cooling decides whether the chip holds high clocks; with insufficient cooling the chip throttles and sustained performance drops. So agent-scenario performance is judged on stable output under long load, not short-term peak.

Apple's M5 Max loses about 8% performance after one continuous hour of AI inference, versus about 22% for x86 platforms under equivalent load, showing its cooling can sustain long full load. Qualcomm's Snapdragon X2 Elite Extreme has a TDP of about 50 W, lowest of the three, and supports fanless designs; lower sustained power helps hold clocks. NVIDIA's RTX Spark has the highest compute density of the three — 1 PetaFLOP in a thin-and-light chassis — and its sustained performance under long full load depends on the thermal design; there's no measured data yet. In my view, this is the most questioned link for RTX Spark right now.
