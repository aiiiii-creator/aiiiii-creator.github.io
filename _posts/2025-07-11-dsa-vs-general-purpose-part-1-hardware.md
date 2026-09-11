---
layout: post
title: "Domain-Specific Accelerators vs. General-Purpose Processors, Part 1: The Hardware"
date: 2025-07-11
categories: [hardware, architecture]
excerpt: "Compute, memory, communication — and where the die area goes. A layer-by-layer comparison of the two design philosophies, and the places where they are already converging."
---

*Compute, memory, communication — and where the die area goes. A layer-by-layer comparison of the two design philosophies, and the places where they are already converging.*

There's no shortage of analysis of general-purpose accelerators, or of one particular domain-specific accelerator. What's rare is a systematic side-by-side of the two routes, layer by layer, from hardware microarchitecture up to the software programming model. This piece is an attempt to fill that gap.

*This is a translation of a piece originally published on Zhihu on July 11, 2025 (last revised April 2026).*

Before getting into the comparison, it's worth stepping back to a larger historical view of this contest. Makimoto's Wave holds that integrated circuits may swing periodically between "general-purpose" and "specialized." We're in the golden decade of DSA acceleration, and at the peak of the cycle. I suspect this may turn out to be an interesting trend to watch.

![Makimoto's Wave](/assets/images/posts/dsa-vs-general-purpose-part-1-hardware/89960ad49b39cc4d1ab8df6af268e170.jpg)

## 1. Hardware

The way I'd summarize today's computer architecture: exploration of software and hardware on three planes — compute, memory, communication — so this piece follows that order. One disclaimer up front: I'm taking the general-purpose and the specialized as two extremes for the sake of exposition, and current products show the two converging.

### 1.1 Compute

#### 1.1.1 The compute array

Within a single core, a general-purpose accelerator raises throughput mainly with large numbers of SIMD units; its emphasis is "parallel uniformity" and scheduling flexibility, suited to large-scale general tasks. A specialized accelerator usually adopts a customized PE architecture inside the compute unit — typically a custom compute-array structure, characterized by its own interconnect and register file. Systolic arrays and the Eyeriss architecture are examples. On top of the array there's usually a general vector accelerator or other function blocks to round out the functionality.

![Compute-array structures](/assets/images/posts/dsa-vs-general-purpose-part-1-hardware/04c6aa746851431ac42efca080866771.jpg)

From a data-dependency angle, the vector accelerator's vector structure is quite general (hence today's assortment of vectorizing compiler optimizations) and can map a wide range of tensor operators efficiently. By contrast, a systolic array or a specialized interconnect is usually optimized for one class of tensor access pattern (local reuse, say), with narrower applicability. In hardware terms, a specific PE array has higher area efficiency and can accelerate compute-bound operators very well.

But for memory-bound operators, **intra-array reuse is nearly useless.** Memory-bound operators lean more on the interaction between the outer memory levels and the nearest cache. So the notion that "a bigger array brings more reuse" is realized mainly through a bigger cache, not simply through array size:

**Intra-array reuse** means that once data enters the PE array, it's used several times over the physical interconnect between PEs, without being re-read from any memory level. This reuse is extremely cheap, because PE-to-PE transfer runs over short wires (typically register-to-register between adjacent PEs) with no arbitration logic or address decoding. A DRAM read costs about 200× a register-file read; an on-chip SRAM read about 6× a register read; a direct PE-to-PE pass is close to register level.

**Extra-array reuse** happens between the array and the outer memory levels. When one tile finishes, the next tile may need to reuse part of the previous tile's data. If that data still sits in an on-chip cache or scratchpad near the array, it needn't be reloaded from DRAM — that's extra-array reuse.

The register file / cache is the main component of area overhead in many compute architectures, determined mainly by its capacity and its bandwidth interconnect. A typical general-purpose accelerator needs at least four ports to satisfy the MAC operation d = a×b + c. A specialized accelerator varies with the array design; the classic systolic array cuts the physical register-port requirement. Though that's not absolute — a MAC-tree structure for GEMV, for instance, has plenty of ports and much higher overall area efficiency than a systolic array, and each PE in the Eyeriss array carries a large register file.

#### 1.1.2 Precision units

Precision choice acts on both critical paths, compute and memory, and directly affects hardware design and data scheduling.

Most specialized accelerators target one precision or a handful. For general-purpose accelerators, precision heterogeneity has become a major problem modern compute architectures must handle. In recent years, tensor operations in different domains have shown markedly different precision needs: signal and image processing commonly use INT8/INT16 to optimize throughput and efficiency; scientific computing and cryptography (NTT, say) need INT32/INT64 for accuracy; machine learning uses INT8/FP16 in inference and FP32/FP64 in training, plus the emerging BF16 format for backpropagation.

So modern vector ISAs (RISC-V Vector, Arm SVE, CUDA) are converging on support for as many as eight precision formats: INT8, INT16, INT32, INT64, BF16, FP16, FP32, FP64. That poses a highly flexible compute-unit design challenge to the microarchitecture.

### 1.2 Memory

Limited off-chip DRAM bandwidth is the system-level bottleneck shared by general-purpose processors and specialized accelerators alike. The mainstream options are DDR and HBM; not covered here. This section focuses on on-chip memory, mainly SRAM.

#### 1.2.1 Data management models

The difference between general-purpose processors and specialized accelerators in on-chip memory is not, at bottom, capacity or number of levels. It's who decides when data is loaded and evicted. That's the key to understanding the generality gap: hardware-managed versus software-managed.

General-purpose accelerators universally use hardware-managed multi-level caches. L1/L2/L3 use hardware replacement policies (LRU and its variants) to decide automatically which data stays on-chip; the programmer doesn't (and can't) precisely control each cache line's lifetime. The core advantage is zero assumptions about access patterns: sequential scan, random jumps, irregular access with complex branches and data dependencies — the cache system adapts automatically through the statistical properties of temporal and spatial locality. The cost is extra hardware: tag storage, comparison logic, replacement-policy state machines, and, in multicore, coherence protocols (MESI/MOESI). GPUs, with their enormous register files (Volta has 256 KB of registers per SM, far larger than its L1), are essentially spending hardware on thread-level data reuse too, hiding memory latency by switching rapidly among many threads.

![Borrowed from a textbook: the GPU's giant register file also serves data reuse](/assets/images/posts/dsa-vs-general-purpose-part-1-hardware/00f669a675aa73a058a57ec1cb4d4dad.jpg)

Specialized accelerators have a simpler memory hierarchy, optimized for a specific task's execution efficiency. AI accelerators favor a software- or compiler-managed scratchpad memory. A scratchpad is directly address-mapped on-chip SRAM: no tags, no replacement policy, no coherence hardware. Data loading and clearing are controlled entirely by the compiler or a hardware scheduler on a pre-computed timeline — the compiler knows at compile time when each matrix tile is needed, how much space it takes, and when it can be overwritten. In image processing, deep-learning inference, and similar domains, multiple compute units (tiles) usually compute in parallel with their own local storage. That design targets highly parallel, regularly accessed workloads. For other workloads (complex random-access patterns), a tile accelerator's storage may be less flexible and less efficient than a multi-level cache. So there's no need for hardware guessing: precise software scheduling gives close to a 100% hit rate, with essentially no cache misses in the traditional sense.

But the dividing line isn't absolute; take sparse workloads. A discussion in the comments raised a valuable question: what happens when the access pattern is no longer fully predictable?

For sparse–dense computation (sparse weights × dense activations), the nonzero positions on the sparse side are irregular but can be determined before the run through CSR/CSC compression. So mainstream sparse NN accelerators (Cambricon-X, SCNN, SparTen) still use "sparse encoding + explicitly managed buffers" — the compiler or hardware controller precomputes access addresses from the sparse index. It's still predictable scheduling.

For sparse–sparse computation (SpGEMM, both operands sparse) it's completely different. The intersection of two sparse matrices' nonzeros can't be determined before the run; the access pattern becomes genuinely runtime-dependent — the same irregular-access problem a general-purpose processor faces. So SpGEMM accelerators (SpArch, OuterSPACE) do need hardware-managed caches or cache-like merge buffers to cope with unpredictable access. That also explains why some accelerators aimed at general tensor computation are starting to adopt multi-level cache structures.

From this angle, the choice of memory-management strategy comes down to how predictable the access pattern is: fully predictable → scratchpad is optimal (high area efficiency, no tag overhead); partly predictable → scratchpad plus a small hardware cache; fully unpredictable → hardware caching is unavoidable. This echoes the convergence trend noted earlier: as accelerators have to support more operator types (sparse, dynamic shapes, conditional execution), the limits of a pure-scratchpad scheme become more apparent, while introducing caching erodes the area and energy advantage specialized accelerators hold over general-purpose processors.

![Borrowed from a paper](/assets/images/posts/dsa-vs-general-purpose-part-1-hardware/010df465036bfefd69a23ce4a9e07efe.jpg)

#### 1.2.2 Memory topology across cores and tiles

The relationship between memory and communication: memory answers "where does the data live," communication answers "how does data get from A to B." They're tightly coupled in design — the partition of the memory hierarchy sets the communication topology's requirements, and communication bandwidth limits in turn constrain memory allocation.

Multicore communication in a general-purpose processor is built on the abstraction of a shared address space. Any core can access any address; the hardware routes the request to the right place and guarantees coherence. Under this model, communication is implicit. "Communication" between two cores is just one core writing an address and another reading the same address, with the cache-coherence protocol (MESI/MOESI) and the on-chip interconnect (NoC or bus) moving data transparently in between. The programmer needn't (and usually can't) specify which physical link data takes or how many hops it makes. That hugely simplifies the programming model, at considerable hardware cost: the coherence protocol has to broadcast invalidations on every write (snooping) or query a directory (directory-based), and that protocol traffic grows with the square or the logarithm of core count, becoming the scaling bottleneck. GPUs are an example: from Volta on, NVIDIA introduced an independent shared-memory datapath within the SM, essentially to bypass the cost of global coherence — shared memory doesn't participate in L2's coherence protocol, so data exchange within an SM is faster and higher-bandwidth.

Multicore communication in a specialized accelerator is built on the abstraction of explicit dataflow. Tiles don't share an address space; there's no coherence protocol; communication is explicit. The compiler or hardware controller generates explicit data-movement instructions specifying source tile, destination tile, data volume, and timing. Under this model, the communication topology, bandwidth allocation, and timing orchestration are all customized by the designer for the target workload.

Despite the diverging routes, the goals are the same: prepare data early (reduce latency), reuse loaded data repeatedly (maximize reuse), and keep compute units highly utilized (reduce idling). The difference is the means: general-purpose processors use automatic hardware mechanisms to cope with memory uncertainty; specialized accelerators use explicit software scheduling to exploit memory determinism. That split also decides the different area allocations — general-purpose processors pour large area into tag storage, coherence logic, and thread-scheduling hardware; specialized accelerators pour the same area into larger local storage and more compute units.

#### 1.2.3 Convergence in memory design, seen from operator optimization

The recent trend toward vectorized and matrixized code is changing the memory characteristics of general-purpose processors, and changing how programmers write operators. Both are worth a closer look.

##### 1.2.3.1 Operator optimization on general-purpose processors

Writing a high-performance operator on a general-purpose processor, the programmer's core task is to maximize the cache hit rate within the hardware-managed cache framework, by adjusting data layout and access order at the algorithm level. The programmer can't directly control what stays in L1, but can make the access pattern match the cache's behavior through a carefully designed tiling strategy.

Take matrix multiplication. A naive triple loop performs terribly on large matrices, because the inner loop's column access of B — if B is row-major — skips a full row width per access, destroying spatial locality and causing massive cache misses. The classic remedy is loop tiling: decompose the M×N×K multiply into small blocks (say 64×64×64) so each block's working set fits fully in L1/L2. Within a block, the programmer further reorders the i/j/k traversal so the innermost loop's accesses are contiguous. Then vectorization — replacing the innermost scalar multiply-accumulate with SIMD instructions (SSE/AVX/NEON/RVV) that handle 4, 8, or 16 elements at a time.

The essence of this process: the programmer is using software to adapt to a hardware mechanism they can't directly control. Cache replacement policy, line size, and associativity are fixed hardware parameters; the programmer can only adjust the access pattern so the cache's automatic behavior happens to line up with the computation. That's why hand-written operators on general-purpose processors concentrate on tiling-factor selection, layout transformation (matrix packing — rearranging sub-matrices into contiguous blocks), and prefetch-instruction placement — all indirect ways of influencing cache behavior. The GEMM implementations in BLAS libraries (OpenBLAS, MKL) run to thousands of lines of assembly, most of the complexity coming from fine adaptation to the multi-level cache hierarchy.

With the arrival of matrix extension instructions (Intel AMX, Arm SME, the RISC-V matrix extensions), general-purpose processors are moving from "vectorized" to "matrixized." These instructions expose two-dimensional tile registers at the ISA level (AMX's eight tmm registers, up to 1 KB each) and perform tile-level matrix multiply-accumulate. From a memory standpoint, tile registers are effectively a small hardware-managed scratchpad — the programmer explicitly loads matrix blocks into tile registers, explicitly triggers the multiply, and explicitly stores the result back. Which means general-purpose processors, on the matrix-compute path, have already partly adopted the specialized accelerator's explicit-management approach.

##### 1.2.3.2 Operator optimization on specialized accelerators

Writing an operator on a specialized accelerator is a completely different mindset. The programmer's (or, more often, the compiler's or toolchain's) task is to precisely orchestrate the movement timing and lifetime of every block of data within an explicitly managed memory hierarchy.

Take a typical systolic-array NPU: the programmer cuts the large matrix into tiles by array size, then explicitly lays out a data-movement plan — in cycle 1, load A's first tile from DRAM into bank0 of the on-chip buffer; in cycle 2, load B's first tile into bank1; in cycle 3, start the systolic array while prefetching the next tile pair into bank2/bank3 (double buffering); in cycle N, write the result from the output buffer back to DRAM. Throughout, the programmer has to know precisely each buffer's size, bank count, and read/write ports, and the DMA channels' bandwidth and latency.

Because all data movement is deterministic and there are no cache misses, compute-unit utilization can in theory approach 100% (unless memory-bound). The price is very poor development effort and portability: change the buffer size, the array size, even the DMA latency, and the whole schedule may need rewriting. That's why specialized accelerators depend so heavily on compilers and automation.

But as the operator types a specialized accelerator has to support keep growing, pure explicit orchestration is under pressure too. Take Transformers: attention's softmax needs a row's sum before normalization (a data dependency crossing one dimension), unlike matmul's regular tiling, so it needs a different buffer strategy. Add dynamic shapes (sequence length known only at inference time), sparse attention masks, and MoE's conditional routing, and less and less is knowable at compile time; the premise of explicit orchestration is eroding. Some modern AI accelerators have begun to add small hardware-managed caches or dynamic scheduling logic for these cases — essentially moving toward the general-purpose processor's approach.

##### 1.2.3.3 The point of convergence

Notably, whichever hardware you write operators on, tiling is the core abstraction. On a general-purpose processor the goal of tiling is to fit the working set into cache; on a specialized accelerator, into scratchpad and array. The mathematical form is identical — strip-mining and reordering of loop nests — and only the constraints differ: the former's come from cache line size, associativity, and the statistics of the replacement policy; the latter's from buffer capacity, bank conflicts, and the deterministic parameters of DMA bandwidth.

That is the compiler-abstraction basis on which MLIR can unify both. MLIR's linalg dialect expresses matrix operations as high-level tensor contractions; a tiling pass decomposes them into sub-problems; then, per target — if it's a GPU, generate a tiled GEMM kernel exploiting shared memory; if it's a systolic-array NPU, generate explicitly buffer-managed DMA scheduling code — different lowerings. The operator's mathematical semantics and tiling structure are shared; only the final memory mapping and synchronization mechanism differ.

### 1.3 Multi-chip scheduling

Multi-card interconnect for general-purpose processors shares one trait — "adapt to the unknown at runtime" — much like their multicore scheduling.

The first pillar is hardware-coherent shared memory (NUMA). Multiple CPU sockets connect over UPI or Infinity Fabric; the hardware maintains cross-socket cache coherence (MESI/MOESI), completely transparent to applications. The second is dynamic OS scheduling. Linux's CFS scheduler does hierarchical load balancing at runtime by CPU topology (SMT → physical core → NUMA node → cross-socket), with work stealing, assigning tasks dynamically with no knowledge of what the workload is. The third is MPI + InfiniBand. MPI provides 115-plus communication primitives (Send/Recv/Allreduce/Alltoall and so on) with no assumptions about the workload — halo exchange in weather simulation, neighbor lists in molecular dynamics, irregular communication in graph analytics, all on the same set of primitives. InfiniBand's RDMA gives sub-microsecond latency and near-line-rate bandwidth over arbitrary topologies.

The specialized accelerator's core is "decide everything at compile time," again like its multicore scheduling. These designs eliminate the runtime overhead of coherence protocols, virtual memory, and dynamic scheduling, achieving deterministic, zero-jitter execution on regular tensor computation. The price is poor workload flexibility: change the input shape and you recompile; data-dependent communication patterns need workarounds; irregular workloads like graph analytics and database queries are fundamentally unsuitable. TPU's XLA compiler, for instance, statically partitions the compute graph across all the chips of the torus topology at compile time, with every DMA transfer pre-scheduled.

### 1.4 Area breakdown

Beyond the three hardware modules above, there's the control section. A general-purpose accelerator's control logic has to cover branch prediction, dynamic scheduling, and other complex logic — it's more state-machine-driven. A specialized accelerator's is simpler.

It's hard to compare a general-purpose processor and a specialized accelerator on different process nodes, so instead we can look at the proportions inside one general-purpose processor. Take a vector processor (VPU): ETH Zürich PULP's Ara. (Mainly because I'm lazy and this was the one dataset I had at hand; I couldn't be bothered to dig up another design…)

![The Ara vector processor](/assets/images/posts/dsa-vs-general-purpose-part-1-hardware/99213ade53b5c5f78e26934a16428160.jpg)

First, the memory structures — the vector register file (VRF) and the queues — are 38.1% of total area. The pure compute portion, because of multi-precision support, is 41.85%. With only four lanes, communication overhead is negligible; the communication units here are mainly the VLSU (Vector Load/Store Unit) and the SLDU (Slide Unit). Communication and control split the remainder. One can more or less extrapolate the three parts' proportions from this small case…

Extending the view to the specialized side, Google's TPU v1 area breakdown is a classic reference: the 256×256 systolic array (Matrix Multiply Unit) is about 24% of the die, the Unified Buffer (24 MB of SRAM) about 29%, and the weight FIFO and other storage about 18% — storage in total is nearly half the area, far more than the compute unit. That confirms a general rule: in memory-intensive specialized accelerators, the biggest consumer of die area is usually not the PE array but the on-chip SRAM. It also explains why the "memory-management strategy" emphasized earlier matters so much for area efficiency — the tag and coherence-logic area a scratchpad saves converts directly into larger effective storage capacity.

Overall, a rough area-allocation pattern emerges. In a general-purpose processor, compute, memory, and control are relatively balanced (roughly 30–40%, 30–40%, 20–30%), because control logic (schedulers, coherence protocols, branch prediction) consumes a great deal of area in exchange for generality. In a specialized accelerator, memory is usually the largest consumer (40–50%), the compute array next (20–30%), and control the smallest (5–15%) — area is allocated as far as possible to modules that directly produce compute or storage value, and control overhead is squeezed to a minimum.

This difference in area allocation essentially reflects every design choice discussed above: general-purpose processors "invest" area in flexibility infrastructure (cache tags, coherence directories, warp schedulers); specialized accelerators "invest" the same area in compute density and data bandwidth. It also echoes Makimoto's Wave: when the application is sufficiently clear, cutting control overhead and enlarging the compute and memory share is the direct source of specialization's efficiency gain; when the applications to be supported diversify, the control share inevitably rises, and the architecture naturally evolves back toward general-purpose.

This piece was finished the day I moved out of the lab, on the way to a new journey. Every point above could be its own article, so the treatment is necessarily incomplete. It's also a look back on my master's years — revisiting old ground to learn something new. Criticism and additions welcome. The next piece compares the software side: hand-written operator optimization and compilers, and hardware–software co-design.
