---
layout: post
title: "PPA for Beginners, Part 1: What a MAC and an SRAM Actually Cost"
date: 2023-05-05
categories: [hardware, ppa]
excerpt: "A first-year master's student's notes from Timeloop on a 40 nm library: the energy and area of an 8-bit versus a 16-bit multiply-accumulate, and why one SRAM read already costs as much as a MAC."
---

*A first-year master's student's notes from Timeloop on a 40 nm library: the energy and area of an 8-bit versus a 16-bit multiply-accumulate, and why one SRAM read already costs as much as a MAC.*

As a first-year master's student and a newcomer to IC design, I've realized I've always focused on architecture and front-end code, and my understanding of the middle and back end is nearly blank. For instance, the core of AI hardware acceleration today is reducing memory accesses, yet I've never once experienced the back-end energy, delay, and area of a multiply-accumulate (MAC) versus a memory — so my research is, frankly, a castle in the air. **I believe that for every engineering student, in academia or industry, concrete parameters are the fertilizer of technique, tightly bound to the innovation and practice we do.**

*This is a translation of a piece originally published on Zhihu on May 5, 2023.*

I've recently been exploring a simulator for AI hardware called Timeloop [1], which comes bound to several advanced process libraries and gives PPA directly from structural parameters. It's a tool that can fill in a lot of rough foundational knowledge about compute, memory, and communication blocks for us beginners (very helpful for forming a first intuition), and turn up some interesting conclusions. So I've summarized it to share. (I'm not very familiar with the simulator or with IC design, so there may be errors — corrections welcome ━(\*｀∀´\*)ノ亻!)

The following is based on a certain 40 nm library.

- **MAC unit**: an 8-bit MAC versus a 16-bit MAC. The area is just that of one 8-bit or 16-bit multiply-accumulator.

![8-bit MAC](/assets/images/posts/ppa-for-beginners-part-1/3ac9165c29e4b987a928932861ab8bac.jpg)

![16-bit MAC](/assets/images/posts/ppa-for-beginners-part-1/2c4786e55833a6a111c696584cdbf0f8.jpg)

Energy per operation and area: 16-bit is 4× 8-bit on both. Standard stuff.

- **A Buffer of the register-file class** (the memory closest to the compute unit) versus **a MainMemory of the SRAM class** (the level above the register file, generally a bit larger). All of the following should be synchronous dual-port SRAM.

> First, a conceptual clarification: a register file and a lot of registers are not the same thing. When we say "register" in IC design we usually mean a D flip-flop, whereas a register file is a kind of memory. [2]

So when do you use a D-flip-flop-based register? An ordinary register uses somewhat more transistors than SRAM and somewhat more power, but its timing is faster and it's more convenient to read and write. See [3] for details.

So the regfile below is also implemented as SRAM.

Below, SRAM arrays of 128×8 and 64×8:

![128×8 SRAM](/assets/images/posts/ppa-for-beginners-part-1/c7caf2d5e286dcf5a8ea67041a49ecb7.jpg)

![64×8 SRAM](/assets/images/posts/ppa-for-beginners-part-1/195384932785bc06470f59bdd29d7261.jpg)

From these, a 16-bit MAC's area falls between the 128×8 and the 64×8 arrays. SRAM array area and access energy growing with size is the normal pattern. Note here that one read from the 64×8 SRAM array already costs nearly as much energy as one 8-bit MAC.

When I switch the SRAM size to 64×16 (word bits still 8), setting the cluster count to 2 reduces area and energy. (In the simulator, the cluster count is mainly tied to the ratio of the configured word bits to the width.) See [4].

![64×16 SRAM, 2 clusters](/assets/images/posts/ppa-for-beginners-part-1/91cb2bd2656ce91650726743eba7b5b2.jpg)

Next up: some loop-tiling knowledge and simulation~

**References**

1. [NVlabs/timeloop](https://github.com/NVlabs/timeloop) — Timeloop performs modeling, mapping and code-generation for Tensor Algebra workloads running on Explicitly-Decoupled Data Orchestration (EDDO) architectures.
2. [Register file vs. SRAM explainer](https://www.elecfans.com/d/2048028.html) (elecfans, Chinese)
3. [What's the difference between a register and a register file?](https://www.zhihu.com/question/580740091/answer/2863336814) (Zhihu, Chinese)
4. [What is the difference between block size and cluster size on a disk](https://www.quora.com/What-is-the-difference-between-block-size-and-cluster-size-on-a-disk) (Quora)
