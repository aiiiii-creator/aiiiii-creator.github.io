---
layout: post
title: "Why Apple's M5 Has Two Kinds of AI Core"
date: 2025-10-31
categories: [hardware, npu]
excerpt: "A Neural Engine and, now, a neural accelerator inside every GPU core. Two conspiracy theories and two real reasons."
---

*A Neural Engine and, now, a neural accelerator inside every GPU core. Two conspiracy theories and two real reasons.*

Apple M5 is Apple's latest chip architecture, built on third-generation 3 nm. As an NPU person, what surprised me was that each GPU core now has a built-in neural accelerator optimized for AI workloads. Plenty of companies have had "dual AI core / multi-tier AI acceleration" concepts before, but either it's hardware with a day job moonlighting at AI compute, or a tensor core added to the GPU. This one delighted me — a chance to pad a post: why, on the same SoC, add a Neural Accelerator inside the GPU on top of the existing Neural Engine?

*This is a translation of a piece originally published on Zhihu on October 31, 2025.*

Apple is famously secretive, so everything below is a guess; additions welcome. First, two conspiracy theories (joking):

1. Like many domestic companies: one vector core, one tensor core, so you can copy NVIDIA's software ecosystem directly (not really).

2. The product of infighting between the company's GPU and NPU teams (not really).

The real reasons, I'd guess:

1. Many tasks need mixed graphics and AI processing, and shuttling data back and forth between the GPU and the Neural Engine every time wastes lots of time and energy — Vision Pro's spatial computing, AI-driven real-time video upscaling, LLM-driven smart NPCs in games, and so on. Many of these tasks aren't mature yet, but the hardware has to pave the way for the software.

2. The Neural Engine's design suits medium-to-high-batch inference, but for small, real-time tasks (local image enhancement, HDR synthesis) the overhead of waking it is too high. A neural accelerator inside the GPU is better suited to giving immediate AI acceleration in low-latency, small-batch scenarios.

Academia rejoices — new hardware-software co-design to do. For someone in my direction, NPUs, the biggest significance is a future chance to intern with Apple's GPU group in Cambridge (I'll work hard to publish more, so some senior person might scoop me up).

Apple Silicon's design philosophy has always been a unified compute architecture — shared memory space, instruction interface, and task-scheduling logic (I'm guessing). Perhaps on that basis, just as CPUs eventually evolved into big and small cores, NPUs are starting their heterogeneous era, splitting by workload?
