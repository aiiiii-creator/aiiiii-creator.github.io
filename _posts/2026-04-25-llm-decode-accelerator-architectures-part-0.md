---
layout: post
title: "LLM Decode Accelerator Architectures, Part 0: The Memory-Bound Setup"
date: 2026-04-25
categories: [hardware, llm]
excerpt: "Why LLM decode is memory-bound, why TPU/GPU-style designs may not fit, and the two main families of dedicated decode accelerators."
---

I just submitted a paper, so I'd like to take this chance to write something
lighter — a technical retrospective on how recent LLM decode accelerators are
actually designed. In the past, when I read these works, I usually came in
chasing one specific question. This time I want a different lens: putting them
on the same diagram from a hardware-architecture perspective and asking what
bottleneck each one targets, why it ended up looking the way it does, and
whether there are deeper common patterns underneath. The exercise is also a way
for me to reorganize my own understanding of the area, so consider this both
a survey and an open question about whether my reading of the field makes sense.

Let me start with how memory-bound LLM decode actually is, and roughly derive
why traditional TPU/GPU-style designs may not be the best fit for it.

## Why decode is memory-bound

When analyzing the hardware bottleneck of LLM decode, we set attention aside
for now — its compute is comparatively small, with the exception of long
context and agentic workloads, which I plan to discuss in a separate post.
That leaves us with the dominant linear operators: the Q / K / V / O
projections, the up / down / gate projections in FFN, and the linear layers
inside MoE experts. All of these can be abstracted as a single matrix
multiplication: input $A$ of shape $M \times K$, weight $B$ of shape
$K \times N$, and output $C$ of shape $M \times N$.

Their arithmetic intensity is

$$
\frac{2MNK}{MK + KN + MN}.
$$

In decode, $M$ is typically much smaller than $N$ and $K$, so the data
movement is dominated by the weight term $KN$. The expression simplifies to

$$
\frac{2MNK}{KN} = 2M,
$$

i.e. roughly $2M$ FLOP/Byte, where $M$ is usually the batch size or some
attention-related parameter.

Compare that with the ridge points of recent accelerators:

| Accelerator | Precision | Ridge point (FLOP/Byte) |
| --- | --- | --- |
| NVIDIA H200 | Dense BF16 | ~206 |
| NVIDIA GB200 / Blackwell (2-GPU superchip) | Dense BF16 | ~313 |
| Google TPU v6e / Trillium | BF16 | ~574 |
| Google TPU 7x / Ironwood | BF16 | ~320 |

You'd need batch sizes in the hundreds before crossing these ridge points —
and that's before factoring in non-linear operators and communication, which
are usually pipeline-hidden but still complicate scheduling. From this angle,
building a dedicated decode accelerator is well-motivated.

## Two routes to feeding the compute units

Looking from the supply path, current decode accelerators are essentially
answering the same question: **how do you efficiently deliver, to the compute
units, the weights and state each token needs?** Along this line, the
mainstream solutions roughly converge into two families:

1. **The SRAM route** — scale up or scale near with SRAM, and design the
   compute hardware around the corresponding dataflow.
2. **The DRAM route** — the same idea, but with DRAM as the supply substrate.

This SRAM / DRAM split is meant only as a coarse observational lens, not a
strict hardware taxonomy. Groq's LPU, for example, is usually understood as
leaning toward large-scale on-chip supply with deterministic execution, but
there has recently been debate over whether their on-chip memory expansion
will go via 3D-SRAM or 3D-DRAM.

A related caveat: many of these systems are not pure decode accelerators.
Cerebras WSE-3 runs both prefill and decode, but its design point reflects
extreme on-chip resources — public materials emphasize 44 GB of on-chip SRAM
and very strong on-chip data supply. SambaNova's RDU combines on-chip SRAM,
HBM, and larger-capacity DDR that all participate in the supply path.

Beyond commercial systems, there is an active academic line of work on
near-memory computing — accelerators sitting on the HBM logic die. I find
this direction quite plausible to land in industry, and Samsung's and
SK Hynix's heavily customized logic dies for HBM4 may already be groundwork
for it. (I've been talking with Mai Lab folks regularly on this, and writing
this up is partly my way of organizing those conversations.)

The next post will start with Cerebras — partly to push myself to keep this
series going.
