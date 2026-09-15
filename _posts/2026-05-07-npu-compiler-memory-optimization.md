---
layout: post
title: "How an NPU Compiler Thinks About Memory"
date: 2026-05-07
categories: [compilers, npu]
excerpt: "Layout, tiling, movement, reuse: the four levers for cutting data movement on a scratchpad machine — and why on an NPU each one is a precondition rather than an optimization."
---

*Layout, tiling, movement, reuse: the four levers for cutting data movement on a scratchpad machine — and why on an NPU each one is a precondition rather than an optimization.*

I've been building an NPU compiler stack lately, and this is my attempt to lay out how memory optimization actually works on one.

*This is a translation of a piece originally published on Zhihu on May 7, 2026.*

An NPU's hardware is far stricter about memory-access patterns than a CPU or GPU. The compute unit is a fixed-shape MAC array (16×16 or 128×128 are typical); the on-chip SRAM is a scratchpad, not a cache, so there is no hardware prefetch and no automatic replacement. Every data transfer has to be an explicit DMA instruction issued by the compiler, and on an NPU all of it has to be settled once, at compile time.

The cost of data movement shows up on two axes at once.

On the time axis: compute on modern accelerators keeps growing faster than bandwidth. Take an NVIDIA H100: about 1000 TFLOPS of FP16 compute against about 3 TB/s of HBM3 bandwidth. Work it through, and for every byte moved into the chip the hardware has to perform roughly 300 floating-point operations on it before the compute units stop idling. That ratio — arithmetic intensity, AI — is the basic metric for judging where an operator's bottleneck is. Most deep-learning operators have an AI far below 300; the time goes to waiting for data.

On the energy axis: moving data costs far more than computing on it. On a mainstream process, reading a 64-bit word from DRAM is about 640 pJ, reading it from on-chip cache about 10 pJ, and an FP32 add about 0.4 pJ. A DRAM access costs more than 1000× the computation. Where an operator spends most of its energy is decided by its AI.

Time and energy share one governing factor: the farther from the compute unit a bit has to travel, the longer it takes and the more it costs. All of an NPU compiler's memory-optimization work reduces to minimizing the time × energy product of data movement over the memory hierarchy the hardware gives you.

There are four basic ways to reduce data movement:

- **Layout**: align the physical arrangement with the hardware's access interface and the DMA burst shape, so every byte moved is a useful one.
- **Tiling**: cut tensors down to what the current memory level can hold, so data is processed close to the compute unit.
- **Movement**: overlap transfers with compute through double buffering, software pipelining, and multiple concurrent DMA engines.
- **Reuse / Fusion**: make every byte brought on-chip do as much work as possible, through operator fusion, loop reordering, and recomputation.

![The four levers. Every one of them is aimed at the same product: the time and the energy it costs to move a byte.](/assets/images/posts/npu-compiler-memory-optimization/01-four-levers.png)

## 1. Layout

Layout is the byte order in which a tensor sits in physical memory. One logical tensor can have many layouts; they are mathematically equivalent, and they can differ by tens of times in hardware access efficiency.

On an NPU the constraints on layout come from two places. First, the compute unit's interface: a fixed-shape MAC array (16×16, 128×128) needs to read a fixed number of values from specific offsets in the scratchpad every cycle, and the layout decides whether those values are physically contiguous. Second, the DMA burst shape: the highest-bandwidth form of a transfer from DDR into scratchpad is a large contiguous block, and the layout decides how much one DMA can move and how many descriptors it takes. A mismatched layout means either the MAC array gets only part of its data each cycle and idles, or the DMA degenerates into many small transfers that never fill the bandwidth.

One example from each operator class.

### GEMM / Linear

Shape (M, K) × (K, N) = (M, N); covers the QKV projections, the FFN, the embedding, and the MoE expert layers of a Transformer.

The most basic layout choice is row-major versus column-major, corresponding to GEMM's NN / NT / TN / TT variants. What NPUs commonly do is pre-pack the matrices further into a block format. Take Ascend's FractalNZ: the weight matrix is cut into 16×16 blocks, stored row-wise inside each block, with the blocks arranged in a predefined two-dimensional order (zN or Nz). This arrangement means that when the Cube unit (a 16×16 MAC array) reads 16×16 values from L0B each cycle, it lands on exactly one contiguous physical block, and the address generator never has to stride.

Weight packing is done at compile time; the model file at runtime already stores FractalNZ, and the DMA pours it straight into the scratchpad. Activations only get their sizes at runtime, so a compiler-inserted reformat operator has to pack them on the fly before each entry into the scratchpad.

### Conv

Shape (N, C, H, W); covers vision models, the visual front end of diffusion models, and the image encoders of multimodal models.

Consider a 16-lane MAC array that consumes the 16 channel values at one (h, w) position per cycle. With NHWC data, the 16 channels of pixel (h, w) are 16 contiguous elements; the MAC reads a contiguous run from one base address and has everything in one cycle. With NCHW, those 16 channels sit at 16 addresses spaced H×W elements apart: either 16 serial reads, cutting throughput to 1/16, or spreading across the SRAM's 16 banks — but a regular large stride usually lands every access in the same bank, and serialization is unavoidable.

![The same tensor and the same MAC array. Only the byte order changed, and with it whether one cycle gets its sixteen values or sixteen.](/assets/images/posts/npu-compiler-memory-optimization/02-nhwc-vs-nchw.png)

### Attention / KV cache

Shape (B, H, S, D); covers Transformer attention.

In training, D usually sits innermost, which suits the chain of GEMMs and vector ops in Q×K^T, softmax, and A×V. D is the head dimension; the reduction inside a head runs along D, and a contiguous layout gives the best vectorization.

In decode, each step generates one new token and has to read all the accumulated K and V against the new token's Q. S grows over time; pre-allocating a large contiguous region wastes memory, while growing on demand causes constant reallocation. vLLM's PagedAttention cuts (B, H, S, D) along S into fixed-size pages (typically 16 or 32 tokens per page); each page is still contiguous internally, and pages are indexed through a block table. That is a design that couples layout with tiling and memory allocation

Layout's failure mode never announces itself. Weights arrive already packed and activations arrive in the framework's native order, so the compiler quietly inserts a reformat to reconcile them. It compiles, the numbers come out right, and you pay a DDR round trip per layer that shows up only as profile time attributed to an operator you never wrote. Upstream MLIR can spell that relayout: `linalg.pack` takes `inner_dims_pos`, `inner_tiles` and `outer_dims_perm`, which is a FractalNZ-shaped blocking in one op. What it cannot do is execute it where you need it. `lowerPack` opens with `// TODO: Support Memref PackOp. Temporarily return failure.`, and `TODO: Support Memref` appears 25 times across `lib/Dialect/Linalg`. The relayout is legible above bufferization and inert below it, which is exactly where the scratchpad lives. — still, at bottom, using layout to match the twin constraints of hardware access and memory allocation.

## 2. Tiling

Tiling means cutting a large tensor into blocks along one or more dimensions and loading only one block at a time into the current memory level. Its root constraint is a simple fact: the whole tensor doesn't fit.

An NPU's memory hierarchy typically runs: off-chip memory (GB), L1 (MB), L0A / L0B / L0C (tens to hundreds of KB), registers (KB). Each level down loses one to three orders of magnitude of capacity and gains bandwidth and access granularity. The intermediate activations of a batch-16 ResNet-50 are tens of MB; the intermediate of a single LLaMA-7B FFN layer is hundreds of MB. None of it fits whole into L1, let alone L0. Tiling isn't an optimization — it's the precondition for data reaching the compute unit at all.

![Each level down loses one to three orders of magnitude of capacity. Nothing of interest fits whole.](/assets/images/posts/npu-compiler-memory-optimization/03-memory-hierarchy.png)

Tile size is set by four constraints together:

1. **Capacity**: the tile's bytes must fit the target level. In GEMM, three tiles (A, B, C) coexist in scratchpad, so Bm·Bk + Bk·Bn + Bm·Bn ≤ L1 capacity.
2. **Hardware granularity**: the tile's size along the MAC array's dimensions must be a multiple of the array edge. On a 16×16 Cube, the M, N, and K dimensions of a tile should all be multiples of 16.
3. **DMA efficiency**: too small a tile makes DMA overhead exceed the transfer itself. Each DMA descriptor has tens to hundreds of cycles of issue latency, so a tile should move at least KB-scale data per DMA to amortize it.
4. **Pipeline overlap**: the tile also has to work with double buffering; when L1 is split in half, the usable capacity halves, and tile size is squeezed further.

The four conflict: hardware granularity wants tiles big, capacity wants them small, DMA efficiency wants them big, double buffering wants them small. The NPU compiler runs an integer search under these constraints for a legal, best-performing set of tile sizes.

![Two constraints push the tile smaller, two push it larger, and the workable band is what survives in between.](/assets/images/posts/npu-compiler-memory-optimization/04-tiling-constraints.png)

Again, one from each class.

### GEMM / Linear

(M, K) × (K, N) → (M, N), with M = K = N = 4096, FP16, 96 MB of data in total. L1 is 1 MB, L0 is 64 KB.

At the L1 level take (Bm, Bn, Bk) = (256, 256, 256); the three tiles use about 384 KB, leaving room for a double buffer. At L0 cut further to (16, 16, 16) to feed the Cube. The whole GEMM decomposes into (M/Bm) × (N/Bn) × (K/Bk) outer L1-tile iterations, each containing (Bm/16) × (Bn/16) × (Bk/16) Cube invocations.

K is the reduction dimension and is handled differently from the parallel ones. Each K-tile's partial sum has to be accumulated. NPUs usually keep the accumulator in L0C and accumulate K-tiles there in place, writing the result out to L1 only once all accumulation is done. This is the Output Stationary idea (more on it in the reuse section) showing up in tiling: the innermost loop of the tile pins the output location and accumulates along K.

### Conv

Shape (N, C, H, W), 3×3 convolution, input (1, 256, 56, 56), weights (256, 256, 3, 3).

Conv has more tile dimensions than GEMM. C is the reduction direction, H and W are spatial, and K (output channels) is another parallel direction. The common cut order is H first (keeping W whole to exploit im2col's contiguous access), then K (output channels are independent, so cutting there introduces no reduction), with C kept as whole as possible in L1 (the reduction runs along C, and cutting it adds partial-sum traffic).

Conv tiles also involve boundary handling. The output tile of a 3×3 conv needs the input tile extended by one ring of padding in H and W — the halo region. The smaller the tile, the larger the halo's share and the more data gets loaded twice. A 56×56 output cut into 7 segments in H is 8 rows each, with a 1-row halo on each side: about 25% redundant loading. That's one source of the lower bound on tile size.

### Attention

Take FlashAttention. The full attention matrix S = Q×K^T has shape (B, H, Sq, Sk); at Sq = Sk = 8192, a single head's S is 128 MB, which fits in no on-chip memory.

FlashAttention cuts Sq into blocks of Bq (typically 64 or 128) and Sk into blocks of Bk (typically the same). Each (Bq, Bk) sub-block computes a partial attention independently: Q_block × K_block^T for a small block of scores, a local softmax, then multiply by V_block for a partial output. Softmax across blocks needs rescaling, so FlashAttention introduces the online-softmax algorithm to let the results from multiple K-tiles merge incrementally, without materializing the full S first.

Landing this tiling on an NPU has extra requirements: Bq × D and Bk × D must fit in L1 together, with room for double buffering, and the softmax's running statistics (per-row max and sum) need their own accumulator space. Ascend and similar chips provide a dedicated vector-unit path for this, running softmax and accumulation in parallel with the Cube.

### The compiler's view

Tile-size search is usually its own pass in an NPU compiler. Input: the compute graph with layouts already assigned, plus a hardware description (capacity per level, MAC size, DMA parameters). Output: the concrete tile shape for each operator at each level.

There are two implementation styles. One is analytic solving on a cost model: list all the constraint equations, take compute × bandwidth utilization as the objective, and solve by integer programming or exhaustive search. That works on regular operators (GEMM, Conv), where the constraints have a closed form. The other is profile-driven autotuning: candidate tile sizes are timed on real hardware and iterated toward the optimum. TVM's auto-scheduler, Triton's autotune, and Huawei CANN's tiling search all belong here.

Tiling fails loudly, and only because somebody wrote the check. Upstream mostly has not. `linalg::promoteSubViews`, the pass that stages a tile into a named memory space, ends its precondition list with `// TODO: Check that the total footprint fits within a given size.` The affine copy generator does carry a budget, and it is instructive about what carrying one gets you: when the buffers exceed `fastMemCapacityBytes` it calls `emitWarning`, returns success, and emits the overflowing code anyway. On a cache that is survivable. On a scratchpad there is no backstop underneath it, so the check that matters is the one you write yourself.

On a GPU, a wrong tile choice usually shows up as lower performance; the run still completes. On an NPU, a tile exceeding L1 capacity fails compilation outright, and a tile violating the MAC divisibility constraint means the Cube won't start. Tiling, like layout, is the textbook case of something going from optimization to precondition.

## 3. Movement

On an NPU, data movement is explicit: every DMA instruction is issued by the compiler, specifying source, destination, shape, and stride. The compute units (Cube, Vector) and the DMA unit are physically independent hardware and can in principle run in parallel. But the default execution order is serial: DMA in, compute, DMA out, next round. Without explicit overlap, the compute unit idles waiting for data and the DMA idles waiting for results.

The goal of movement optimization is to run the two concurrently and hide transfer latency behind compute.

### Double buffering

The most basic technique is double buffering, also called ping-pong. Split the L1 region holding a tensor in two (buffer A and buffer B) and alternate transfer and compute between them.

Three steady-state moments:

T0: DMA moves tile 0 into buffer A; the compute unit is idle. T1: DMA moves tile 1 into buffer B; the compute unit processes tile 0 in buffer A. T2: DMA moves tile 2 into buffer A; the compute unit processes tile 1 in buffer B.

In steady state, compute and DMA fully overlap; total time is max(compute time, transfer time) rather than their sum. If compute time is at least the DMA time, the DMA is fully hidden, and vice versa.

![Steady-state double buffering. The cost of the overlap is half of L1, which is where the tile-size constraint comes from.](/assets/images/posts/npu-compiler-memory-optimization/05-double-buffering.png)

The price is halved L1 capacity. That is where the "pipeline overlap" constraint on tile size in the previous section comes from: a tile can't fill L1; it has to leave room for its twin.

### Software pipelining

One computation often involves several transfers. In GEMM, each outer tile iteration has four steps: move the A tile into L1, move the B tile into L1, the Cube computes the C tile (accumulating in L0C), move C out of L1 once accumulation completes. The four can be scheduled independently.

Software pipelining interleaves different stages of multiple tile iterations. The longest single action sets the total throughput. Pipeline depth grows from a 2-stage double buffer to 4 stages or more, and the number of buffer copies grows with it: L1 gets split into 4 or even 8 parts, each holding one tile.

Software pipelining on an NPU isn't implicit. On CPUs and GPUs the compiler leans on out-of-order execution units and scoreboards to reorder instructions automatically; an NPU compiler has to generate the interleaved sequence of transfer and compute instructions explicitly, and insert a synchronization instruction (semaphore, event) at every dependency boundary. One wrong sync point and you get a data race or a deadlock.

### Multiple DMA engines in parallel

NPUs usually have several independent DMA paths, each dedicated to a different leg of the route. On Ascend these are the MTE (Memory Transfer Engine) pipes, split by direction: one brings data from off-chip memory into the on-chip buffers, another stages it from L1 down into the L0 operand buffers, and another writes results back out. The engines are physically independent and can work simultaneously.

That hardware structure makes multi-level pipelining possible. While one tile sits in L0 being consumed by the Cube, the next tile moves from L1 into L0, the one after that moves from off-chip memory into L1, and the previous round's output drains out of the accumulator. Each leg is a different engine, so none of them waits on another.

![Four tiles in flight at once, each on its own engine. The compiler has to emit one instruction stream per engine and every barrier between them.](/assets/images/posts/npu-compiler-memory-optimization/06-mte-pipeline.png) Every segment of the pipeline has its own transfer hardware; none blocks another.

The price is that the compiler has to generate a separate instruction stream for each engine and make sure the cross-engine dependencies are expressed correctly through synchronization primitives. In Ascend's ISA, MTE instructions and Cube instructions belong to different pipes, and pipes synchronize through barriers.

### Asynchronous DMA and prefetch

Some NPUs offer fire-and-forget asynchronous DMA: issue the transfer and return immediately, so the compiler can hoist transfer instructions as early as possible to overlap with later compute. On GPUs, cp.async (Ampere onward) and TMA (Hopper onward) are the same kind of mechanism.

Prefetch is a special case of asynchronous DMA: predict the data the next round needs and start the transfer early. On regular operators like GEMM the prediction is accurate; on operators with dynamic shapes or complex control flow it can miss, and the data moved goes unused, wasting bandwidth.

### Coupling with tiling

Movement isn't an independent optimization layer; its feasibility is entirely determined by tiling.

- Double buffering needs two copies of each tensor in L1: the tile capacity budget halves.
- Four-stage software pipelining needs four copies: the budget drops to a quarter.
- The deeper the pipeline, the higher the hardware concurrency, but the tighter the limit on tile size. Too small a tile and the DMA issue overhead exceeds the transfer, and the pipeline can't hide it either.

There's an optimal band for tile size: too small and DMA issue overhead dominates; too large and the pipeline is too shallow to use the hardware's concurrency. The band is usually between KB and tens of KB, set jointly by DMA issue latency, bandwidth, and compute density.

### The compiler's view

In an NPU compiler, movement optimization shows up as instruction scheduling. Input: the operator compute graph with layout and tiling done. Output: an instruction sequence with pipe assignments and sync instructions.

The scheduling algorithm is typically list scheduling or modulo scheduling. The former greedily schedules instructions one by one in dependency order — simple, not necessarily optimal. The latter targets loop structures, unrolling the loop into software-pipelined form and solving for the minimum initiation interval (II); on regular operators it can approach the hardware's concurrency ceiling.

Get movement wrong and nothing tells you either. One missing barrier between a transfer pipe and the compute pipe hands the array a half-written buffer: wrong numbers, non-deterministically, passing at test shapes and failing at production ones. A lost wakeup hangs the card instead. Upstream supplies more mechanism here than its reputation suggests. `transform.loop.pipeline` ships a target-independent modulo scheduler that assigns cycles by dependence and wraps them modulo the II, and it composes with `memref::multiBuffer` for the buffer expansion. What it does not ship is a latency model. The long latency is attached to vector transfers, a copy op is invisible to it, and `memref.dma_start` is created by the affine lowering and consumed by nothing downstream. The skew is free. Every cycle count and every wait is yours.

A GPU compiler leans on hardware out-of-order execution to hide most scheduling problems, and software pipelining on a GPU counts as performance tuning. On an NPU there is no hardware arbitration between pipes; software pipelining is the main determinant of throughput, and the compiler's scheduling quality directly decides whether an operator runs at 30% or 90% of nominal compute.

## 4. Reuse / Fusion

Everything so far has been about moving data faster. Reuse is the other direction: how much work can one byte do once it's on-chip?

The metric is arithmetic intensity (AI), defined as the operator's total floating-point operations divided by the bytes read from the next level up. As noted at the start, the H100's roofline knee is around 300; an operator with an AI below that is bandwidth-bound, and only above it can the compute units saturate. The numbers differ on an NPU; the structure is the same.

There are three basic ways to raise AI: operator fusion so intermediates never touch DDR, loop reordering so data is reused across the innermost loop, and recomputation to trade compute for memory.

### Operator fusion

A deep-learning model is a chain of many small operators. A typical LayerNorm, expanded to its mathematical definition, is seven or eight element-wise and reduction operators in series (subtract the mean, square, variance, add epsilon, sqrt, divide, multiply by gamma, add beta). Executed one at a time, each step reads the tensor from DDR, computes, and writes back to DDR; the tensor gets shuttled more than ten times.

Fused, the chain becomes one kernel: read the tensor from DDR once, keep every intermediate in registers or L1, write the result back to DDR once. AI goes up an order of magnitude, and DDR traffic drops to 1/N.

![Unfused, the tensor crosses DDR after every operator. Fused, it crosses twice.](/assets/images/posts/npu-compiler-memory-optimization/08-fusion.png)

Fusion boundaries are set by analysis of the compute graph. Typical fusible patterns:

- **Element-wise chains**: several consecutive element-wise operators (add, mul, relu, gelu) merged into one kernel — the simplest kind of fusion.
- **GEMM + epilogue**: the GEMM output feeds straight into bias, activation, or a residual add, and these trailing element-wise ops complete in place in the GEMM's L0C accumulator, with no write to L1 and read back.
- **Reduction + element-wise**: the LayerNorm / Softmax pattern of reduce first, then element-wise scale.
- **Producer–consumer**: one operator's output is immediately consumed by the next, with matching shapes.

Typically not fusible: a shape change between operators (a reshape crossing the reduction dimension), a control-flow fork (the output consumed by several downstream operators at different times), or a device boundary (the tensor sharded across multiple NPUs).

FlashAttention is the landmark example of fusion. Plain attention consists of Q×K^T, scale, softmax, and ×V; the naive implementation writes out the full intermediate S from Q×K^T (128 MB at 8K×8K, as computed in the previous section) and reads it back. FlashAttention fuses the whole sequence inside the tile loop: each (Bq, Bk) sub-block's partial score is softmax-localized as soon as it's computed, multiplied by V, and accumulated into the output — S never materializes. AI rises by more than an order of magnitude, and DDR traffic falls by a factor of N/Bk.

### Loop reordering

For one GEMM, `C[i,j] += A[i,k] * B[k,j]`, the different orderings of the three loops (i, j, k) correspond to different reuse patterns.

- **k innermost**: fix (i, j), accumulate along k. C[i, j] stays in a register accumulating while A[i, k] and B[k, j] stream past. That's Output Stationary; each output value is reused K times.
- **j innermost**: fix (i, k), stream along j. A[i, k] stays in a register while B[k, j] and C[i, j] stream past. Each A value is reused N times.
- **i innermost**: fix (k, j), stream along i. B[k, j] stays in a register while A[i, k] and C[i, j] stream past. Each B value is reused M times.

![Which operand stays in the register decides which one gets the reuse. Software calls it loop order; a systolic array calls it dataflow.](/assets/images/posts/npu-compiler-memory-optimization/07-stationarity.png)

Which value stays put in a register decides which operand gets the highest reuse. On CPUs and GPUs this is a software choice of loop order; on a systolic array the same decision is welded into the hardware as the Output Stationary, Weight Stationary, and Input Stationary dataflows. It is the same reuse problem — software solves it by reordering loops, hardware solves it with the PE interconnect.

Real systems pick stationarity independently at each memory level. A typical GPU GEMM: OS at the register level (the output tile accumulates in registers), WS + IS at the shared-memory level (both weights and inputs cached), no stationarity at HBM (streamed). NPUs are similar: WS inside the systolic array (weights poured into the PEs and held), OS at L0C (partial sums accumulate), streaming from L1 and DDR.

### Recomputation

Recomputation (also called gradient checkpointing) is the reverse move: don't save the intermediates; recompute them when needed.

Backpropagation in training needs the forward pass's intermediate activations to compute gradients. A standard 7B Transformer at sequence length 4096 and batch 1 has activations totaling on the order of ten GB; longer sequences or bigger batches quickly exceed a single card's memory. Saving everything means enormous volumes of data read and written repeatedly on HBM.

Recomputation saves only the activation at each layer's entry, and on the backward pass re-runs the forward from that entry to regenerate the intermediates. The cost is doing the forward twice (once forward, once again during backward); the gain is activation memory down by one to two orders of magnitude.

Recomputation looks like the opposite of raising reuse, but it's the other pole of the same trade-off: when the cost of storage (moving data plus occupying capacity) exceeds the compute cost of regenerating it, discarding and recomputing is cheaper in total. In LLM training, where activations are enormous, almost every modern framework (PyTorch's checkpoint, Megatron's selective recomputation) has recomputation on by default.

### Coupling with layout and tiling

Reuse optimization is deeply coupled with the two sections before it.

Fusion requires the fused operators to share a tile partition. If two operators' tile shapes disagree, the fused kernel needs extra internal transfers or reshuffles, which can lower AI instead of raising it. The compiler has to do tile-consistency analysis before deciding on fusion.

The optimal loop order is tied directly to layout. The dimension the innermost loop walks should be the innermost dimension of the layout, or you get strided access. A K-innermost loop order over NCHW gives contiguous cross-channel access; the same order over NHWC gives cross-pixel strides and lower efficiency. Once the layout is fixed, the feasible set of loop orders narrows.

Recomputation's relationship with tiling shows up in re-doing tiles on the backward pass: if the forward was tiled as (Bm, Bn), the backward re-does it with the same partition, and each tile's intermediates are regenerated, consumed, and discarded independently, without raising peak memory.

### The compiler's view

Reuse optimization is done by two kinds of passes in an NPU compiler.

The first is graph-level operator fusion. The compiler recognizes fusible patterns in the operator graph (element-wise chains, GEMM + epilogue, reduction + scale), merges several operators into a fused op, and the back end generates a single kernel for it. TVM's FuseOps, MLIR's linalg fusion, and Huawei CANN's fusion engine are all of this kind.

Over-fusion is the failure with negative value. The fused chain's live set stops fitting L1, so either allocation fails outright or the compiler inserts reshuffles that cost more than the DDR round trip fusion was supposed to save. Nothing upstream stops you: the elementwise fusion pass's entire profitability policy is `producer->hasOneUse()`. The tile-and-fuse hook is better informed, since it sees the `tensor.extract_slice` and could compute a footprint from it, but the shipped default accepts every candidate. The passes that decide fusion have no model of a transfer at all, so the decision cannot see the byte budget it is spending.

The second is loop-level schedule transformation. When generating the concrete kernel, the compiler reorders (interchange), merges (fuse), and blocks (tile — sharing the underlying mechanism with section 2) the operator's loop nest to produce the instruction sequence with the best stationarity. Polyhedral compilation is the theoretical foundation; TVM's schedule, Halide's schedule, and MLIR's affine dialect are all built on the polyhedral model.

A GPU compiler does both kinds of pass too, but on a GPU reuse optimization is more of a performance amplifier: skip fusion and it runs slowly, but it runs. On an NPU, fusion is often the precondition for running at all — an unfused kernel can fail compilation because an intermediate tensor exceeds L1 capacity. It's the same optimization-to-precondition phenomenon we saw with tiling and movement.

Read the four failures in order and they escalate: a silent performance tax, a compile error, non-deterministic corruption, an optimization with negative value.

| Lever | On an NPU (software-managed scratchpad) | On a GPU (hardware-arbitrated cache) |
| --- | --- | --- |
| Layout | silent reformat op, correct numbers, a DDR round trip per layer | slightly worse coalescing |
| Tiling | tile exceeds L1 and compilation fails outright | lower occupancy, still runs |
| Movement | missing barrier, non-deterministic corruption or a hung card | hardware arbitrates, still correct |
| Reuse / Fusion | live set exceeds L1, allocation fails or fusion runs slower | one extra kernel launch |
 That ordering is not a coincidence, and neither is where upstream MLIR stops helping. Above bufferization it is genuinely strong; `linalg.pack` states what a MAC-port layout is, the tiling drivers cut and fuse an iteration space, and none of it is worth rewriting. Below bufferization, where the scratchpad and the explicit transfer and the byte budget actually live, coverage thins to plumbing. The generic DMA op carries one stride level and no consumer; the multi-dimensional descriptor types that exist are vendor-scoped, in NVGPU and AMDGPU, templates to copy rather than infrastructure to reuse. That line is not a maturity gap. It marks where hardware stops arbitrating. A GPU compiler can hand the memory system its problem and be right often enough. An NPU compiler cannot, so everything below the line got built by whoever needed it, one target at a time.
