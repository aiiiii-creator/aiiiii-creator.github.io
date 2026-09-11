---
layout: post
title: "Was DeepSeek V4 Trained on Ascend?"
date: 2026-04-25 12:00:00
categories: [hardware, llm]
excerpt: "Probably not — at least not the pre-training. What the technical report actually says about hardware, why FP4 training is so hard on the datapath, and where domestic chips still fall short."
---

*Probably not — at least not the pre-training. What the technical report actually says about hardware, why FP4 training is so hard on the datapath, and where domestic chips still fall short.*

The title is clickbait, I'll admit — I don't actually know. I can only analyze from the technical side, and record what I learned along the way.

*This is a translation of a piece originally published on Zhihu on April 25, 2026 (revised May 10).*

## 1. Is there evidence?

After DeepSeek V4's release came a long string of good news for domestic chips: Day-0 support from eight vendors, Huawei Ascend as the first to support it, and, for the first time, a technical report listing a Huawei NPU and an NVIDIA GPU in the same hardware-validation table. Together with the earnings coming out of the West Coast, this drove a big rally in Hong Kong semiconductor stocks over those few days.

But read each headline carefully and it carries the same qualifier: inference. Inference and training are two different things, and within training, pre-training and continued training are two more. What chips V4-Pro's main pre-training used, DeepSeek didn't say. That's a stark contrast with the detail in the December 2024 V3 technical report — "2,048 H800s, about 54 days, $5.576M" (which is also what shocked US stocks last year). The training code Ascend open-sourced alongside corresponds to V4-Flash's continued training, not pre-training from scratch. Continued training can run on a few hundred cards for a few days; pre-training needs a ten-thousand-card cluster running continuously for months, with requirements on interconnect bandwidth, long-run stability, and fault recovery on a different level entirely.

So the three tasks are a staircase of difficulty: inference easiest, continued training in the middle, pre-training hardest. Domestic chips' progress is stuck on that staircase — inference passed, continued training just reached, pre-training unknown.

The only explicit statement about hardware in the technical report is in §3.1:

> "We validated the fine-grained EP scheme on both NVIDIA GPUs and Huawei Ascend NPU platforms."

Plus the rather contentious explicit mention of "Ascend," in the small print of the V4-Pro API pricing page:

> Constrained by high-end compute, Pro's serving throughput is currently very limited; the Pro price is expected to drop substantially once the Ascend 950 supernode ships in volume in the second half of the year.

All that's known for now: MoE routed-expert weights in FP4; attention, embedding, LM head, and non-MoE dense layers in FP8; activations in FP8; optimizer state in FP32; gradients in FP8, accumulated into FP32 master weights. The report states explicitly that "on current hardware, FP4×FP8 has the same peak throughput as FP8×FP8; future native-FP4 hardware could bring another 1/3 efficiency gain" — a line that maps directly onto the Ascend 950 series' MXFP4 2 PFLOPS spec. FP4 happens to be the precision the Huawei Ascend 950PR supports, which opens the possibility of domestic-chip training later on.

I've been working on a datapath lately, and the microarchitecture here really is a pain. FP4 (E2M1) has only 4 bits, so only 16 representable values — ±0.5, ±1, ±1.5, ±2, ±3, ±4, ±6, and zero. Against FP8's 256 values, resolution and dynamic range both collapse, and that's the source of all the trouble.

In the FP8 era, per-tensor scaling (one FP32 scale shared by the whole tensor) was enough, but on FP4 it fails outright: 16 values is too coarse, and the moment the value distribution inside a tensor is slightly uneven, either lots of values underflow to zero or the outliers saturate. OCP's Microscaling (MX) format is the current answer: every 32 elements share one 8-bit E8M0 exponent scale, pulling dynamic range back to ~10^38 (FP32 class), but within the block it's still 4-bit resolution. Dynamic range solved; resolution not.

Trickier still is the asymmetry between forward and backward in training. A backward pass involves three matrix multiplies: fprop (activation × weight), dgrad (gradient × weightᵀ), wgrad (activationᵀ × gradient). dgrad's numerical distribution has a severe outlier tail — after gradients pass back through all the amplifying and attenuating factors, some positions sit orders of magnitude above the mean, and FP4 can't hold on dgrad; precision simply collapses. Blackwell's answer is an asymmetric configuration: MXFP4 for fprop, MXFP6 or MXFP8 for dgrad. That means the hardware has to support mixed bit widths — the Tensor Core has to switch precision, the datapath's accumulator width is designed for the worst case, and the scheduler and scale-decode logic both get more complex. If a chip only built a single MXFP4 4-bit unit, either training won't converge or dgrad falls back to BF16 and eats half the dividend.

The most lethal factor for training is accumulator precision. A matrix multiply is K partial products accumulated into one accumulator; in theory the relative error of K floating-point accumulations scales as √K × ε, with ε the accumulator's machine precision. With FP4 inputs, each partial product carries only about 8 bits of information, adding roughly 2× accumulation noise and doubling the demand on accumulator mantissa width. The DeepSeek V3 paper states explicitly that the H800's FP8 accumulator has only FP22-equivalent precision (13 mantissa bits vs. FP32's 23); at K = 4096 the maximum relative error approaches 2%, and V3 had to move partial sums to CUDA Cores for another FP32 accumulation every 128 WGMMAs, sacrificing 10–20% throughput for stability. An FP22 accumulator running MXFP4 would have error several times worse than FP8, and V3's "move it to the vector core" trick can't rescue it in the FP4 era — the hardware has to natively support a true FP32 accumulator, or the precision loss can't be patched in software. Personally, I think Blackwell widening its FP4 GEMM accumulator to true FP32 is the hidden key to why B200 training works.

Other vendors have disclosed little. AMD's MI300X has been independently confirmed to have true full-precision FP32 accumulation, but domestic chips have disclosed almost nothing here. DeepSeek V3.1 introduced the UE8M0 scale factor and said, in a pinned comment on its official account, that it was "designed for the next generation of domestic chips" — is that a door left open?

## 2. Why probably not

Which is to say, there's no tight chain of logic showing it was trained on Ascend. And not just Ascend — this is the weakness of every domestic chip.

As usual, start with the performance gap:

![Spec comparison (Ascend vs. NVIDIA)](/assets/images/posts/was-deepseek-v4-trained-on-ascend/756314a9aed09c9cefbc23c88398425e.jpg)

Because LLM training is heavily compute-bound, it needs far more cards than inference — often clusters of thousands or tens of thousands. That requires two things. First, interconnect bandwidth. Pre-training isn't a single-card affair; a ten-thousand-card cluster synchronizes gradients constantly, with an all-reduce every step. Huawei's HCCS is 784 GB/s per direction, about 40% of NVLink 5. On the same training task, domestic cards spend a higher share of time waiting for data, and compute utilization can't rise. That's pre-training's biggest hardware shortfall.

Second, long-run stability. Ten thousand cards for months: any card failing, any network hiccup, any numerical overflow in a layer, and the whole job may roll back or be lost. Meta's Llama 3 405B training, for example, ran 16,384 H100s for 54 days, averaging a component failure every 3 hours — 419 in total — yet kept effective training time above 90%, with checkpoint-plus-recovery costing only 2.1%. That figure is the industry benchmark. Getting there takes hardware–software coordination, operations, scheduling, and fault recovery ground in together, and the grinding means stepping on one landmine after another. NVIDIA's stack came out of one large-scale training run after another; every hole has been filled. The domestic side has no such accumulated experience.

Third, numerical stability. Training in low precision (FP8, or experimentally FP4) saves half the memory and compute, but low precision overflows easily; across a few hundred layers a signal can amplify thousands of times and the model collapses. Holding it steady takes work on both the algorithm and the hardware — the hardware has to guarantee enough accumulation precision, and the algorithm has to add constraints (V4's mHC does exactly this). Inference is far more forgiving: BF16 or even INT8 runs fine, and a little precision loss doesn't matter.

Finally, the software stack. CUDA's two decades of software ecosystem, profiling tools, checkpoint mechanisms, and fault-recovery tooling are the most important moat. The TPU's software team, for instance, is larger than its hardware team, and much of it is devoted to training. CANN has been catching up fast over the last two years, and TileLang, TorchTitan-NPU, and AutoFuse are all filling gaps; V4's continued training running at all shows this toolchain reaching a stage of maturity. But pre-training — long-cycle, large-scale, sensitive to long-tail failures — depends most on toolchain maturity, and that is harder to catch up on than the hardware itself.

At bottom it's a chicken-and-egg problem. Nobody has done trillion-scale pre-training on domestic cards, so the engineering experience can't accumulate; without that experience, nobody dares put their flagship model up as the first to try. So who fires the first shot, and when, matters far more than the hardware parameters.

## 3. Hardware–software adaptation

That said, the technical report has plenty of interesting points, and they also read as software optimization for domestic-chip training and inference.

### 3.1 mHC

Residual connections accumulate and amplify signals through hundreds of layers. The original Hyper-Connections (expanding a single residual into 4 parallel streams, mixed between layers by a 4×4 matrix M) increased expressiveness, but an unconstrained M produced 3000× signal amplification in the 27B ablation (paper data) — which means a 1.6T training run would almost certainly collapse.

mHC (Manifold-Constrained Hyper-Connections; Liang Wenfeng's own arXiv paper, January 2026) has one core innovation: forcing M to be doubly stochastic (every row and column sums to 1, all entries non-negative), equivalent to constraining M to the Birkhoff polytope, which geometrically guarantees "signal conservation" and forces the spectral norm ≤ 1. Each training step runs 20 Sinkhorn-Knopp iterations (alternating row/column normalization) to push the constraint error to 1e-5, cutting signal amplification from 3000× to 1.6× — three orders of magnitude of stability.

mHC pushing signal amplification down three orders of magnitude is essentially using algorithmic stability to compensate for the hardware layer's gap in fault tolerance — which I read as leaving a door open for domestic-chip training.

### 3.2 Muon

The industry-standard AdamW optimizer keeps two FP32 states per parameter (first moment m, second moment v); for a 1.6T-parameter model that is 12.8 TB of optimizer state alone, several times the model weights.

Muon stores no gradient statistics; instead it orthogonalizes the gradient matrix G itself (pulling all singular values toward 1 so the update magnitude is balanced across orthogonal directions), approximating orthogonality in 5 steps with the Newton-Schulz iteration polynomial `aX + bX(XᵀX) + cX(XᵀX)²` (coefficients a = 3.4445, b = −4.7750, c = 2.0315, the optimal third-order coefficients Keller Jordan found numerically).

V4 uses a Muon + AdamW hybrid:

- Muon handles the 2D matrix parameters: hidden layers, attention projections, MoE experts.
- AdamW still handles: embedding (a lookup table, no meaningful orthogonalization), the LM head (sensitive to output distribution), RMSNorm (scalar parameters), and mHC's static mixing matrix (already Birkhoff-constrained).

Muon keeps no m or v, only a single FP32 momentum, saving about 6.4 TB of memory on the backbone.

Ascend 950 will presumably only be able to source HBM from CXMT going forward, with relatively small capacity and bandwidth: 144 GB of HBM per card, matching the H200's 141 GB but behind the state-of-the-art B300's 288 GB. Even with ZeRO-1 spreading optimizer state across 1024 cards, each card still carries 12.5 GB of AdamW state — add model shards, activations, KV, and communication buffers and you're near the HBM ceiling. Muon cuts that per-card item to 6.25 GB, directly freeing HBM headroom for longer sequences, larger batches, and sparser EP partitions at the same card count.

### 3.3 CSA + HCA

Standard Transformer self-attention's O(N²) compute and O(N) KV-cache memory both explode at N = 1M, so a single card's HBM can't hold the KV and compute-unit utilization drops.

No MLA this time; V4 resolves it with two complementary paths:

- **CSA (Compressed Sparse Attention)**: first compress every m = 4 consecutive tokens' KV into one entry with softmax-gated pooling, then use an FP4-precision Lightning Indexer to pick the top-k most relevant compressed blocks for each query (k = 1024 for V4-Pro, 512 for V4-Flash), keeping an uncompressed sliding window over the most recent 128 tokens.
- **HCA (Heavily Compressed Attention)**: the reverse — compress a 1M sequence into about 7,800 entries at an aggressive m′ = 128, then do full dense attention.

The two are stacked alternately along the depth of the network; effective complexity drops to O(N·k/m + N·w) ≈ N × 384 — asymptotically linear, cutting attention FLOPs at 1M context by 2500× versus dense and the KV-cache footprint to 10% of V3.2's (paper data). CSA/HCA's compression ratio cuts cross-node KV transfer to 1/10 of V3.2's, directly compensating for the bandwidth gap. And the full KV of a 1M context fits in a single 8×910C node, so cross-node communication doesn't dominate latency.
