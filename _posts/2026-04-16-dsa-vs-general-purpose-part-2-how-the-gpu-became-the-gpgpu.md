---
layout: post
title: "Domain-Specific Accelerators vs. General-Purpose Processors, Part 2: How the GPU Became the GPGPU (I)"
date: 2026-04-16
categories: [hardware, gpu]
excerpt: "Every CUDA origin story skips a chapter: G80. Not a general-compute bolt-on to an old GPU but a demolition of the graphics pipeline — and it only worked because 90 nm arrived exactly when it did."
---

*Every CUDA origin story skips a chapter: G80. Not a general-compute bolt-on to an old GPU but a demolition of the graphics pipeline — and it only worked because 90 nm arrived exactly when it did.*

In April 2026, NVIDIA's market cap returned to $4.85 trillion. Since its 1999 IPO its market cap has risen more than 850,000×. In semiconductors that's a number with almost no reference point. The mainstream narrative of how NVIDIA got here has long been settled: it won on the ecosystem, it won on CUDA. There are already plenty of technical articles analyzing CUDA's API design, nvcc's compiler optimizations, the hand-written kernel tricks in cuDNN and cuBLAS, and so on.

*This is a translation of a piece originally published on Zhihu on April 16, 2026.*

But there's a chapter this script usually skips: the 2006 chip codenamed G80, the GeForce 8800, which in most retrospectives is just the CUDA launch platform, mentioned in passing. It wasn't an old GPU with a general-compute "add-on"; it tore down almost the entire graphics pipeline and replaced the underlying compute substrate. Few articles discuss how the hardware architecture evolved, and few step back to ask a more basic question: without G80's few key hardware changes, could the CUDA ecosystem have grown at all?

## 1. The GPU era

### 1.1 The fixed-function pipeline

The earliest GPUs (the 1999 GeForce 256, and the earlier 3dfx Voodoo series) essentially "cast" the OpenGL/DirectX graphics API directly into silicon. Every triangle on screen went from vertex coordinates to final pixel color through a fixed processing chain:

![The fixed-function graphics pipeline](/assets/images/posts/dsa-vs-general-purpose-part-2-how-the-gpu-became-the-gpgpu/a58fdaefc1fa3c4e6998ac72db7562b1.jpg)

![](/assets/images/posts/dsa-vs-general-purpose-part-2-how-the-gpu-became-the-gpgpu/fc2da337a696310e9be51a6a6cf9fa16.jpg)

Every stage of this pipeline was hard-wired special-purpose circuitry. "Programming" meant developers configuring a few parameters of those circuits through API calls — which texture, Phong or Gouraud lighting, alpha blend or additive blending. The circuits' behavior itself was fixed. That design was extremely efficient for graphics, since every stage was deeply optimized for its one operation. But from a general-compute viewpoint it was nearly useless: you couldn't make the rasterizer run an FFT, or the texture unit do a matrix multiply. The hardware wasn't a compute resource; it was the physical embodiment of a graphics algorithm.

Take NVIDIA's G70 (the 2005 GeForce 7800 GTX) and ATI's R580 (the 2006 Radeon X1900) as representative: a typical GPU was several functionally heterogeneous special-purpose units strung together.

The **vertex shader (VS)** handled triangle vertex coordinate transforms — projecting points from 3D space onto the 2D screen. G70 had 8 VS units. Its hardware was optimized for "little data, lots of floating point": operating on hundreds of thousands to millions of vertices, each needing matrix multiplies and vector normalization, so VS units generally had full 32-bit floating-point ALUs.

The **pixel shader (PS)** computed a color for every rasterized pixel — texture lookups, lighting, alpha blending. G70 had 24 PS units, three times the VS count. That ratio wasn't arbitrary: a frame has far more pixels than vertices (one triangle can cover thousands of pixels), so the PS needed more hardware parallelism. The price was lower compute precision than the VS — it handled color, so FP16 or even fixed point was enough, with no use for full IEEE 754 precision. So in the same generation, VS floating point was "good but few," PS floating point "poor but many," and the two ALUs weren't the same thing in hardware at all.

The **geometry shader (GS)**, the third programmable shader, introduced by DirectX 10 and OpenGL 3.2, arrived several years after VS and PS (VS/PS existed from DirectX 8; the GS only with DirectX 10 in 2006). It sits in the graphics pipeline between the VS and rasterization.

The **texture mapping unit (TMU)** read textures from video memory. Note the word: read. The TMU was read-only, with a strictly limited access pattern: you could only tell it "I want the texel at texture coordinate (u, v)," and the hardware did the address calculation, cache lookup, and bilinear/trilinear filtering. The TMU's entire premise was "texture data is a read-only resource."

The **raster operations unit (ROP)** wrote the PS's pixel colors back to the framebuffer, handling depth test, stencil test, and alpha blending. The ROP was the only write path from the PS to video memory, and the write location was strictly determined by the coordinates of the pixel being rendered — a pixel couldn't go write some unrelated location in the framebuffer.

The memory subsystem was hardware-partitioned by graphics resource type: textures through the texture cache (read-only, optimized for 2D locality), vertices through the vertex buffer (streaming reads), the framebuffer owned by the ROPs. These partitions barely communicated, with isolated address spaces.

### 1.2 Programmable shaders

The 2001 GeForce 3 was the first turning point, introducing the programmable vertex shader; the 2002 Radeon 9700 and the 2003 GeForce FX added the pixel shader. This step mattered — for the first time the GPU had hardware units that could "run programs."

But that "programmable" came heavily discounted:

- The instruction set was extremely limited. Early vertex shaders supported only a few dozen vector instructions — add, multiply, MAC — and no loops or branches (or only fixed-count unrolled loops).
- Precision was graphics-grade. Pixel shaders for a long time used FX12, FP16, FP24 — "good enough" floating-point formats that didn't remotely meet IEEE 754: rounding, special values (NaN/Inf), and overflow behavior were all vendor-defined.
- Vertex and pixel shaders were two separate pipelines on the chip, with different register counts, instruction caches, and execution units. That caused load imbalance.
- The old GPU's memory-access model was built entirely around texture sampling: shaders could read from memory (via the texture unit, with hardware filtering and caching) but could almost never write to arbitrary locations. The pixel shader's output was fixed: it could only write to the pixel position the rasterizer assigned it, never "elsewhere." Fine in the graphics world (pixel A shouldn't overwrite pixel B).

### 1.3 The scheduling model

Pre-G80 GPU scheduling was classic wide SIMD: one instruction driving a group (typically 4-wide or wider) of parallel datapaths, matching graphics' RGBA four channels or a 2×2 pixel quad. That mode ran graphics smoothly — every pixel does the same thing, with few branches. But general compute is full of data-dependent branches (if-else, while), and wide SIMD hitting a branch either forces synchronization or idles some lanes; efficiency collapses fast.

## 2. The start of generality

The GeForce 8800 GTX, released in November 2006 with the G80 chip: 681M transistors, 128 stream processors, 155 W TDP. By external spec it was still a "graphics card" — it could run Crysis, it output DirectX 10 graphics. But open its die shot and the internal structure had almost no inheritance from the previous G70. The changes below correspond to the generality problems of the previous section.

### 2.1 Why generality became necessary

First, because the graphics API itself changed first: DirectX 10, which Microsoft shipped with Windows Vista in 2006, mandated at the API level that all shaders (VS, PS, GS) use exactly the same instruction set, access the same resources, and support the same data types. That was a hard constraint on GPU vendors: either the API layer fakes unification while the hardware stays split — AMD's R600 (Radeon HD 2900) nominally unified its hardware units but was internally still 5-wide VLIW, and the scheduler struggled to fill those five slots — or the API and the hardware unify together: since the same instruction set has to be supported anyway, merge the hardware units and let the scheduler allocate at runtime. That's the origin of G80's unified shaders.

The way game complexity grew also drove pre-G80's "fixed ratio" hardware design into a dead end. 2001–2003 (the run-up to Half-Life 2 and Far Cry): scene geometry relatively simple, lots of compute in per-pixel lighting, texturing, and shader effects. PS pressure far exceeded VS, and vendors kept raising the PS:VS ratio — G70 set it at 24:8, three to one. 2004–2006 (the run-up to Oblivion and Crysis): games chased big worlds, dense vegetation, high-poly characters, and geometric complexity exploded. Post-processing (HDR, bloom, depth of field) got heavier too. The VS/PS load ratio swung wildly between games and even between scenes in one game. Whatever PS:VS ratio you chose, a large share of hardware sat idle.

Coincidentally, before G80 a small band of academics was already trying general compute on GPUs. It traces to about 2002 — Stanford's Ian Buck (later CUDA's father) started the Brook for GPUs project then, trying to map general compute onto GPUs with a stream programming model. Other schools had similar groups. Their papers were almost all about "what trick we used to get around the GPU's limits" — storing data in textures, computing by rendering a quad, branching via depth test. It yielded interesting applications: early GPU ray tracing, GPU finite-element simulation, GPU-accelerated molecular dynamics. Inside NVIDIA, people had been tracking this work closely since about 2003. Buck graduated from Stanford in 2004 and went straight to NVIDIA, where he led the CUDA project.

There was also a process-level reason. G80 was NVIDIA's first flagship on **TSMC 90 nm**; against G70's 110 nm, transistors per unit area rose about 65%. That gave the architects an "extra budget" — they could add the hardware that "graphics doesn't need but general compute does" without significantly growing the die:

- Full IEEE 754 floating-point units (larger than the simplified versions)
- Shared memory and register files (16 KB of SRAM per SM)
- Load/store units and address-generation logic (absent from the graphics pipeline)
- A more complex scheduler (managing dynamic warp allocation)

Together these took roughly 30–40% of G80's transistor budget — on 110 nm, either the die size runs away (cost and yield both suffer) or graphics performance has to be cut. 90 nm let NVIDIA **have it both ways**: graphics performance still doubled while a whole set of general-compute hardware went in. Had G80 been delayed two years to 65 nm, the CUDA ecosystem might never have started — the 2007–2008 window of GPGPU research would have been missed. Had it been done two years earlier on 110 nm, G80 would have been a big, expensive monster that couldn't beat the competition. **The 90 nm timing, that node, is what made the "radical rewrite" commercially viable.** This sort of "just in time" looks like Jensen Huang's vision in hindsight; in the moment it had a considerable element of luck.

Beyond that, the graphics market's ceiling was in sight and the general-processor market would be bigger. Not elaborated here.

### 2.2 G80's hardware changes

The GeForce 8800 GTX, November 2006, G80: 681M transistors, 128 stream processors, 155 W TDP. By external spec still a "graphics card" — runs Crysis, outputs DirectX 10. But the die shot shows almost no inheritance from G70.

**Unified shader architecture.** G80's most visible and most fundamental change was merging the VS, PS, and GS — three physically separate shader-unit types — into one kind of general "streaming processor" (SP). G80 had 128 SPs, each a scalar floating-point ALU that could run vertex, pixel, or geometry programs.

**IEEE 754.** Each SP had a full IEEE 754-1985-compliant single-precision FP ALU supporting add, multiply, MAD, reciprocal, and reciprocal square root, handling NaN, Inf, denormals, and rounding modes strictly per the standard. The entire numerical ecosystem of computing — LAPACK, FFTW, MATLAB, NumPy, every numerical-methods textbook — is built on IEEE 754's assumptions. Once GPU floating-point results could match the CPU bit for bit (at least in single precision), researchers could verify GPU code against a CPU reference, reproduce others' published numerical results, and confidently ship GPU-accelerated versions to production.

**Global memory model.** G80 for the first time let GPU programs read and write arbitrary addresses — no longer hard-partitioned into texture/vertex/framebuffer as pre-G80, but video memory as one uniformly addressed linear space. The SP instruction set gained general `load addr` and `store addr value` instructions, and scatter (writing to different locations) and gather (reading from different locations) became one ordinary instruction each. This change directly unlocked general parallel programming. `a[i] = b[j] + c[k]`, the most ordinary statement imaginable, needed render-to-texture ping-pong to fake on pre-G80 and became one line of CUDA on G80. Reduction, scan, sort, hash maps — the basic parallel primitives all became writable.

**Shared memory and the SM.** G80's 128 SPs weren't laid flat across the chip; they were grouped in 8 groups of 16 SPs, each a streaming multiprocessor (SM). Each SM had, beyond its 16 SPs, several key pieces of cooperative hardware:

- 16 KB of shared memory — programmer-managed on-chip SRAM (a "software-controlled cache," or scratchpad) readable and writable by all threads in the SM, with a few cycles of latency, two orders of magnitude faster than video memory.
- The `__syncthreads()` barrier — a hardware sync primitive within the SM guaranteeing all threads in a block reach a point before continuing.
- The Cooperative Thread Array (CTA, CUDA's thread block) — a group of threads bound to one SM, able to communicate through shared memory and synchronize through the barrier.

**SIMT execution and warp scheduling.** SIMD is "one instruction operating on multiple data," and the programmer sees a "vector" — you write code worrying about "I'm operating on a 4-wide or 8-wide vector." SIMT is "one instruction driving multiple threads," and the programmer sees a "scalar" — your kernel code is entirely scalar-style, as if writing one thread's logic, while the hardware packs 32 threads into a **warp** sharing one PC. That hugely lowers programming difficulty; there's no need to understand the vector hardware underneath.

Next: how today's NVIDIA GPGPU architecture evolved from there.
