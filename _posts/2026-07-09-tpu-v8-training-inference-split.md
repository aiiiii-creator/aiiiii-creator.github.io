---
layout: post
title: "TPU v8: Google Splits Training and Inference Back Into Two Chips"
date: 2026-07-09
categories: [hardware, tpu]
excerpt: "Google's eighth-generation TPU splits training and inference back into two chips, 8t and 8i. A layer-by-layer teardown of how they take opposite trade-offs from the compute core all the way to packaging and the host."
---

<style>
.tf-fig{ margin:2.4rem 0; }
.tf-card{ border:1px solid rgba(29,27,24,0.12); background:#fffdf9; border-radius:18px; padding:20px 20px 14px; box-shadow:0 20px 60px rgba(38,32,24,0.08); }
.tf-title{ font-family:"Space Grotesk","Noto Sans SC",sans-serif; font-weight:700; font-size:1rem; color:#1d1b18; margin-bottom:4px; letter-spacing:-0.01em; }
.tf-sub{ font-family:"Noto Sans SC",sans-serif; font-size:.85rem; color:#625a50; line-height:1.45; margin-bottom:10px; }
.tf-note{ font-family:"Noto Sans SC",sans-serif; font-size:.78rem; color:#8a8175; margin-top:8px; }
.tf-cap{ font-family:"Noto Sans SC",sans-serif; font-size:.86rem; font-style:italic; color:#625a50; text-align:center; margin-top:12px; line-height:1.5; }
.tf-svg{ width:100%; height:auto; display:block; }
.tf-svg .grid{ stroke:rgba(29,27,24,0.10); stroke-width:1; }
.tf-svg .axis{ stroke:rgba(29,27,24,0.35); stroke-width:1.5; }
.tf-svg .ridge{ stroke:#b4361a; stroke-width:1.5; stroke-dasharray:5 4; }
.tf-svg .ridgelab{ fill:#b4361a; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:700; }
.tf-svg .dline{ stroke:#0f766e; stroke-width:2.5; fill:none; stroke-linejoin:round; stroke-linecap:round; }
.tf-svg .darea{ fill:#0f766e; opacity:.12; }
.tf-svg .dot{ fill:#0f766e; stroke:#fffdf9; stroke-width:1.5; }
.tf-svg .vlab{ fill:#1d1b18; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:600; text-anchor:middle; }
.tf-svg .tick{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; }
.tf-svg .atitle{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:600; }
.tf-svg .ptitle{ fill:#1d1b18; font-family:"Space Grotesk","Noto Sans SC",sans-serif; font-size:14px; font-weight:700; }
.tf-svg .node{ fill:#fffdf9; stroke:#0b5f59; stroke-width:1.5; }
.tf-svg .nodeb{ fill:#f4efe6; stroke:rgba(29,27,24,0.30); stroke-width:1.2; opacity:.6; }
.tf-svg .lk{ stroke:rgba(29,27,24,0.32); stroke-width:1.4; fill:none; }
.tf-svg .lkw{ stroke:rgba(29,27,24,0.30); stroke-width:1.2; stroke-dasharray:4 3; fill:none; }
.tf-svg .lkf{ stroke:#0b5f59; stroke-width:3; fill:none; opacity:.5; }
.tf-svg .flow{ stroke:#0f766e; stroke-width:2; fill:none; }
.tf-svg .farrow{ fill:#0f766e; }
.tf-svg .lbl{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; }
.tf-svg .lblm{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; text-anchor:middle; }
.tf-svg .divider{ stroke:rgba(29,27,24,0.12); stroke-width:1; }
.tf-svg .sx{ fill:#b4361a; font-family:"Noto Sans SC",sans-serif; font-size:13px; font-weight:700; text-anchor:middle; }
.tf-svg .pill{ fill:#0f766e; opacity:.16; }
.tf-svg .pilltext{ fill:#0b5f59; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:700; text-anchor:middle; }
.post-content table{ display:block; overflow-x:auto; border-collapse:collapse; font-family:"Noto Sans SC",sans-serif; font-size:.9rem; }
.post-content th,.post-content td{ border-bottom:1px solid rgba(29,27,24,0.12); padding:9px 12px; text-align:left; vertical-align:top; }
.post-content thead th{ border-bottom:2px solid rgba(29,27,24,0.25); font-weight:700; white-space:nowrap; }
.post-content blockquote{ border-left:3px solid #0f766e; }
</style>

On April 22, 2026, at the Google Cloud Next '26 keynote, Google unveiled its eighth-generation TPU and separated training and inference back into distinct silicon. The plant-based naming line set by v6 Trillium and v7 Ironwood was dropped, and the market names return to plain numbers with a suffix: **TPU 8t** (codename Sunfish, co-designed with Broadcom) for training, and **TPU 8i** (codename Zebrafish, co-designed with MediaTek) for inference.

The eighth generation returns to the v5e/v5p style of forking the two workloads. After two generations of unified SKUs, Trillium (v6e) and Ironwood (v7), Google is again treating the economics and architectural requirements of training and inference as diverging rather than converging.

Here is the full specification comparison across the two chips and Ironwood, in two parts.

| Dimension | TPU 8t (Sunfish, training) | TPU 8i (Zebrafish, inference) | TPU v7 Ironwood |
|---|---|---|---|
| Design partner [reported] | Broadcom | MediaTek (first time) | Broadcom |
| Process [reported] | TSMC 2nm-class (reported, not official) | TSMC 2nm-class (reported) | TSMC N3P |
| Packaging [reported] | 2 compute dies + 1 IO die + auxiliary chiplets + 6× HBM3E | 1 compute die + 1 IO die + 1 CAE chiplet die + 6× HBM3E | 2 compute dies (chiplet) |
| TensorCores / die | 2 (1 per chiplet) | 2 | 1 |
| Dedicated accelerator [official] | SparseCore (4, 2 per chiplet) + "LLM Decoder Engine" (new, not detailed) | CAE (1, standalone chiplet die), physically replacing the 4 SparseCores | SparseCore × 4 |
| HBM capacity [official] | 216 GB HBM3E (12-Hi, 6 stacks × 36 GB) | 288 GB HBM3E | 192 GB HBM3E (8-Hi) |
| HBM bandwidth [official] | 6,528 GB/s (≈6.5 TB/s), below Ironwood | 8,601 GB/s (≈8.6 TB/s, 1.32× of 8t) | 7,370 GB/s |

| Dimension | TPU 8t (training) | TPU 8i (inference) | TPU v7 Ironwood |
|---|---|---|---|
| On-chip SRAM (VMEM) [official] | 128 MB (64 MB × 2 chiplets) | 384 MB (3× Ironwood / 8t) | 128 MB |
| FP4 peak / chip [official] | 12.6 PFLOPS (native, new) | 10.1 PFLOPS (native) | no native FP4 |
| FP8 peak / chip [inferred] | ~6.3 PFLOPS (inferred) | ~5.0 PFLOPS (inferred) | 4.614 PFLOPS (first native) |
| Bytes/FLOP @ FP4 | 0.52 | 0.85 (notably higher) | n/a |
| ICI scale-up bandwidth [official] | 19.2 Tb/s bidirectional (2× Ironwood) | 19.2 Tb/s bidirectional | ~9.6 Tb/s |
| Scale-out DCN / chip [official] | 400 Gbps (4× previous gen) | not separately disclosed (internal to Boardfly) | — |
| Host CPU [official] | Axion ARM | Axion ARM (double CPU hosts per server) | x86 |
| TPU:CPU ratio [reported] | 2:1 | 2:1 | historically 4:1 |
| Liquid cooling [official] | 4th-gen CDU | 4th-gen CDU | 3rd gen |

<p class="tf-note">Source tags in the original table: HBM figures and FP4 peaks via The Register; SRAM and host-CPU details via 9to5Google; ICI bandwidth via HyperFRAME Research; scale-out DCN via Data Center Dynamics. Process node and packaging are reported, not officially confirmed.</p>

## Two Workloads, Two Points on the Roofline

Training batches are homogeneous. All samples enter the same forward, backward, and parameter-update loop, follow identical compute paths, and share one set of weights per layer. The training step is a static compute graph. Data parallelism spreads the batch across thousands of chips; each chip's local micro-batch is small, but the effective global batch is thousands times the micro-batch, reaching millions of tokens. That number sets training's arithmetic intensity. Once a weight byte is read onto the chip, millions of tokens touch it across three passes, forward, backward dx, and backward dW, with reuse often above a few thousand. The training step's arithmetic intensity settles in the thousands of FLOP/byte, above the ridge point of any modern accelerator, so the bottleneck falls on compute.

Inference batches are heterogeneous. A single batch can hold three users running a prefill chunk and twenty-seven users running single-token decode, each request differing in length and generation progress. One iteration later the batch shape can be different: finished requests leave, newly arrived requests enter. **Continuous batching**, the scheduling technique at the core of vLLM, Orca, and SARATHI, exists to handle this dynamic shape. Larger batches also raise arithmetic intensity, but the mechanism and the ceiling differ from training.

Batching B users' decode together lets the model weights be shared across them. Weights are read once and used by all users, which multiplies decode's 1 FLOP/byte by B. KV cache cannot be shared; each user reads its own. The arithmetic-intensity curve therefore runs about 1 FLOP/byte at batch=1, 7 at batch=8, 25 at batch=32, 45 at batch=64, 75 at batch=128, and 110 at batch=256. That still sits below the H100's ridge AI of 295, because the KV cache term grows linearly with batch and never leaves the denominator. This is the structural difference from training: training arithmetic intensity rises without bound as batch grows, while inference batching lifts arithmetic intensity toward a finite asymptote.

<figure class="tf-fig">
  <div class="tf-card">
    <div class="tf-title">Inference arithmetic intensity vs. decode batch size</div>
    <div class="tf-sub">The curve climbs toward but never crosses the H100 ridge point of 295 FLOP/byte.</div>
    <svg class="tf-svg" viewBox="0 0 660 360" role="img" aria-label="Line chart: arithmetic intensity versus decode batch size, staying below the H100 ridge of 295">
      <line class="grid" x1="60" y1="30" x2="560" y2="30" />
      <line class="grid" x1="60" y1="120" x2="560" y2="120" />
      <line class="grid" x1="60" y1="210" x2="560" y2="210" />
      <line class="axis" x1="60" y1="300" x2="560" y2="300" />
      <text class="tick" x="52" y="34" text-anchor="end">300</text>
      <text class="tick" x="52" y="124" text-anchor="end">200</text>
      <text class="tick" x="52" y="214" text-anchor="end">100</text>
      <text class="tick" x="52" y="304" text-anchor="end">0</text>
      <line class="ridge" x1="60" y1="34.5" x2="560" y2="34.5" />
      <text class="ridgelab" x="310" y="25" text-anchor="middle">H100 ridge point = 295</text>
      <polygon class="darea" points="60,299.1 160,293.7 260,277.5 360,259.5 460,232.5 560,201 560,300 60,300" />
      <polyline class="dline" points="60,299.1 160,293.7 260,277.5 360,259.5 460,232.5 560,201" />
      <circle class="dot" cx="60" cy="299.1" r="4.5" />
      <circle class="dot" cx="160" cy="293.7" r="4.5" />
      <circle class="dot" cx="260" cy="277.5" r="4.5" />
      <circle class="dot" cx="360" cy="259.5" r="4.5" />
      <circle class="dot" cx="460" cy="232.5" r="4.5" />
      <circle class="dot" cx="560" cy="201" r="4.5" />
      <text class="vlab" x="60" y="291">1</text>
      <text class="vlab" x="160" y="285">7</text>
      <text class="vlab" x="260" y="269">25</text>
      <text class="vlab" x="360" y="251">45</text>
      <text class="vlab" x="460" y="224">75</text>
      <text class="vlab" x="560" y="192">110</text>
      <text class="tick" x="60" y="320" text-anchor="middle">1</text>
      <text class="tick" x="160" y="320" text-anchor="middle">8</text>
      <text class="tick" x="260" y="320" text-anchor="middle">32</text>
      <text class="tick" x="360" y="320" text-anchor="middle">64</text>
      <text class="tick" x="460" y="320" text-anchor="middle">128</text>
      <text class="tick" x="560" y="320" text-anchor="middle">256</text>
      <text class="atitle" x="310" y="346" text-anchor="middle">decode batch size</text>
      <text class="atitle" x="16" y="165" text-anchor="middle" transform="rotate(-90 16 165)">arithmetic intensity (FLOP/byte)</text>
    </svg>
    <div class="tf-note">Values from the analysis below; the KV-cache term keeps the asymptote finite.</div>
  </div>
  <figcaption class="tf-cap">Figure 1. Inference arithmetic intensity as a function of decode batch size, with the H100 ridge point marked as a horizontal asymptote the curve never reaches.</figcaption>
</figure>

KV cache capacity caps batch more directly. Llama-3 70B under GQA-8 needs 320 KB of KV per token, so batch 256 at 8K sequence consumes 671 GB. One H100 80GB cannot hold it, TPU v5p at 95GB cannot hold it, and even the TPU 8i's 288GB needs three chips. Long context is worse: batch=8 at 128K sequence needs 335 GB. Inference batch size is bounded by the HBM capacity left after model weights, not chosen freely. Training barely faces this constraint, because ZeRO or FSDP spread optimizer states across thousands of chips, leaving each chip enough HBM for activations.

Inference carries a hard constraint absent from training: the latency SLO. TPOT must stay within tens of milliseconds, so waiting for 256 users to arrive before forming a batch is not an option, since the earliest user would already have timed out. Cloud inference relies on continuous batching to pack work dynamically: batches reassemble at iteration granularity, requests enter and leave continuously, and batch contents change hundreds of times per second.

Public benchmark numbers confirm this regime.

| Platform | Model | Batch | Utilization |
|---|---|---|---|
| TPU v5e-8 (JetStream) | Gemma 7B | 32 to 64 | MBU ~60% |
| TPU v5p-256 | PaLM 540B (decode-heavy) | 16 to 32 per replica | MFU ~30% |
| Ironwood pod | Gemini Pro | 64 to 128 | MBU up to 70% |
| H100 ×8 NVLink | Llama-3 70B | 64 to 256 | MBU 60% to 80% |

The v5e single card has only 16 GB HBM, but multi-chip TP spreads weights and KV cache across a v5e-8 slice totaling 128 GB, still serving mid-size models at high batch. Google's "v5e is 3.5× cheaper than v5p" inference cost figure rests on this many-small-chip deployment mode.

Stringing these numbers together produces a counterintuitive result. Multi-batch cloud inference relieves the memory-bound condition without actually pushing the workload to the compute-bound side. Below batch 100 the regime is purely **memory-bound**, and the goal is to raise HBM bandwidth utilization from under 5% at batch=1 to 60% to 80%, with TPOT set directly by HBM bandwidth. Above batch 300 the workload could in theory cross the ridge into compute-bound, but almost no one runs it there, because KV cache capacity and the TPOT SLO have already capped batch. Groq is the exception: its pure-SRAM architecture gives very high effective bandwidth and a very low ridge AI, so its decode sits naturally on the compute plateau, at the cost of model-size flexibility.

> Inference batching lifts a single request's 1 FLOP/byte to tens or over a hundred FLOP/byte and MBU from under 5% to 60% to 80%, but it never crosses the ridge point. It parks the workload in the high-efficiency zone of the memory-bound regime.

Training uses data parallelism to push the effective batch to millions of tokens and parks arithmetic intensity on the compute plateau. Both workloads use batching, but they land at different points on the roofline. Training hardware therefore optimizes compute, and inference hardware optimizes HBM bandwidth and capacity.

## The Compute Core: Block-Scale Multiply Inside the MX Unit

"Training uses BF16/FP32" is a pre-2023 paradigm. Three developments established that frontier-scale pretraining entered an FP8-dominant, FP4-experimental era from 2024: NVIDIA's H100 introduced native FP8 (E4M3 for forward, E5M2 for backward); DeepSeek-V3 ran a full FP8 pretraining across 671B parameters and 14.8T tokens; and that run's loss curve nearly matched the BF16 baseline. By 2026, training precision has settled into a layered mix. Forward and backward GEMMs run FP8 and FP4, over 90% of total FLOPs, while attention accumulators, LayerNorm, and optimizer states keep BF16/FP32 as a fallback.

FP8 splits into two formats because forward and backward training place different demands on numerical distribution. E4M3 (4-bit exponent, 3-bit mantissa) carries one more mantissa bit, giving higher precision but a narrower dynamic range, suited to the relatively concentrated weight and activation distributions in the forward pass. E5M2 (5-bit exponent, 2-bit mantissa) carries one more exponent bit, giving a wider magnitude span but lower precision, matching the long-tailed gradient distribution in the backward pass. This division of labor carried from H100 to Ironwood and is the de facto standard. FP4 halves the bit width again, which helps compute, memory access, and communication.

Both chips carry 2 TensorCores and a native FP4 datapath, but 8t fuses **block-scale multiply** into the MXU pipeline. FP4 training depends on the **microscaling** format: a single 4-bit number has only 16 representable values, so a block-shared scale factor is required to pull the effective dynamic range back to the FP16 level. For comparison, OCP's MXFP4 uses a 32-element block with an E8M0 shared scale (only powers of two). NVIDIA's NVFP4 takes another route: a 16-element block, a scale in E4M3 (a true FP8), plus a second-level per-tensor FP32 scale, trading two-level scaling for finer local dynamic range. These are not similar paths; the choice of format is itself a trade-off between scale-handling cost and precision loss.

Google has not published the specific FP4 numeric format, but it can be inferred from the compute unit. NVIDIA's Tensor Core is a MAC-tree-like structure: a set of parallel FP4 multipliers feed products into a reduction tree, completing accumulation and clipping at the end of a short dot-product path. This fits NVFP4's 16-element block, since the dot-product parallel width aligns with the block boundary and the scale needs one multiply at the end of the reduction tree, at low hardware cost. The systolic-array dataflow is different. One output element comes from a reduction across the full tile edge (typically 128 to 256); the mantissa accumulates stage by stage along the dataflow, and the scale factor must be injected into the correct PE in step with the mantissa. Copying NVFP4's 16-element block directly would require switching scale 8 to 16 times within each systolic row, raising the cost of the broadcast network and PE registers, and MXFP4's 32-element block eases part of this but still does not align. The reasonable engineering move is to align the block length to the tile dimension (one scale per edge), or to use two-level blocking: an outer level synchronized to the tile edge to remove frequent scale switching, and an inner level keeping finer microscaling for numerical precision. Both options require a custom format.

<figure class="tf-fig">
  <div class="tf-card">
    <svg class="tf-svg" viewBox="0 0 660 300" role="img" aria-label="MAC-tree reduction versus systolic-array reduction, showing where the scale factor is applied">
      <defs>
        <marker id="tf-mflow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
          <path class="farrow" d="M0,0 L6,3 L0,6 Z" />
        </marker>
      </defs>
      <line class="divider" x1="330" y1="36" x2="330" y2="272" />
      <text class="ptitle" x="24" y="24">MAC-tree Tensor Core (NVIDIA)</text>
      <text class="ptitle" x="354" y="24">Systolic array (TPU MXU)</text>
      <g>
        <rect class="node" x="32" y="48" width="16" height="16" rx="3" /><rect class="node" x="68" y="48" width="16" height="16" rx="3" />
        <rect class="node" x="104" y="48" width="16" height="16" rx="3" /><rect class="node" x="140" y="48" width="16" height="16" rx="3" />
        <rect class="node" x="176" y="48" width="16" height="16" rx="3" /><rect class="node" x="212" y="48" width="16" height="16" rx="3" />
        <rect class="node" x="248" y="48" width="16" height="16" rx="3" /><rect class="node" x="284" y="48" width="16" height="16" rx="3" />
        <line class="lk" x1="40" y1="64" x2="58" y2="104" /><line class="lk" x1="76" y1="64" x2="58" y2="104" />
        <line class="lk" x1="112" y1="64" x2="130" y2="104" /><line class="lk" x1="148" y1="64" x2="130" y2="104" />
        <line class="lk" x1="184" y1="64" x2="202" y2="104" /><line class="lk" x1="220" y1="64" x2="202" y2="104" />
        <line class="lk" x1="256" y1="64" x2="274" y2="104" /><line class="lk" x1="292" y1="64" x2="274" y2="104" />
        <circle class="node" cx="58" cy="110" r="6" /><circle class="node" cx="130" cy="110" r="6" />
        <circle class="node" cx="202" cy="110" r="6" /><circle class="node" cx="274" cy="110" r="6" />
        <line class="lk" x1="58" y1="116" x2="94" y2="152" /><line class="lk" x1="130" y1="116" x2="94" y2="152" />
        <line class="lk" x1="202" y1="116" x2="238" y2="152" /><line class="lk" x1="274" y1="116" x2="238" y2="152" />
        <circle class="node" cx="94" cy="158" r="6" /><circle class="node" cx="238" cy="158" r="6" />
        <line class="lk" x1="94" y1="164" x2="166" y2="200" /><line class="lk" x1="238" y1="164" x2="166" y2="200" />
        <circle class="node" cx="166" cy="206" r="6" />
        <rect class="pill" x="118" y="222" width="96" height="24" rx="12" />
        <text class="pilltext" x="166" y="238">× block scale</text>
        <text class="lblm" x="166" y="266">one scale multiply, at the tree output</text>
      </g>
      <g>
        <line class="flow" x1="372" y1="52" x2="516" y2="52" marker-end="url(#tf-mflow)" />
        <text class="lbl" x="372" y="44">mantissa accumulates along the row</text>
        <text class="sx" x="391" y="72">×</text><text class="sx" x="425" y="72">×</text>
        <text class="sx" x="459" y="72">×</text><text class="sx" x="493" y="72">×</text>
        <rect class="node" x="380" y="78" width="22" height="22" rx="3" /><rect class="node" x="414" y="78" width="22" height="22" rx="3" />
        <rect class="node" x="448" y="78" width="22" height="22" rx="3" /><rect class="node" x="482" y="78" width="22" height="22" rx="3" />
        <rect class="node" x="380" y="112" width="22" height="22" rx="3" /><rect class="node" x="414" y="112" width="22" height="22" rx="3" />
        <rect class="node" x="448" y="112" width="22" height="22" rx="3" /><rect class="node" x="482" y="112" width="22" height="22" rx="3" />
        <rect class="node" x="380" y="146" width="22" height="22" rx="3" /><rect class="node" x="414" y="146" width="22" height="22" rx="3" />
        <rect class="node" x="448" y="146" width="22" height="22" rx="3" /><rect class="node" x="482" y="146" width="22" height="22" rx="3" />
        <rect class="node" x="380" y="180" width="22" height="22" rx="3" /><rect class="node" x="414" y="180" width="22" height="22" rx="3" />
        <rect class="node" x="448" y="180" width="22" height="22" rx="3" /><rect class="node" x="482" y="180" width="22" height="22" rx="3" />
        <text class="lblm" x="447" y="230">scale switched 8–16× per row,</text>
        <text class="lblm" x="447" y="246">in step with the mantissa</text>
      </g>
    </svg>
  </div>
  <figcaption class="tf-cap">Figure 2. Dot-product reduction in a MAC-tree Tensor Core (scale applied once at the tree output, aligned to a 16-element NVFP4 block) versus long reduction along a systolic array (scale injected in step with the mantissa along the datapath).</figcaption>
</figure>

Where the scale factor is handled sets pipeline efficiency. Ironwood was native FP8 with software-emulated FP4, so the FP4 path had to take a fallback route: the VPU dequantizes FP4 to BF16, the MXU runs the GEMM, and the VPU requantizes back to FP4 in the epilogue. The VPU blocks once at the head and once at the tail of every GEMM, thinning **systolic-array** utilization. 8t folds the dequantize, GEMM, requantize three-stage pipeline into the MXU at once, letting the block-scale multiply and the mantissa multiply-accumulate happen in the same units at the same time. Which MXU stage absorbs the scale (fused inside each PE, inlined in the epilogue, or a separate broadcast channel) is the other side of the format question: the format sets how often the scale appears, and the dataflow sets where the scale is injected, so the two must be co-designed.

Amin Vahdat's words on this in the keynote were:

> redefined performance capability by moving block scale multiplication directly inside the MX units… delivering nearly three times the compute performance per pod

That 3× needs unpacking. 8t's single-chip FP4 at 12.6 PFLOPS against Ironwood's single-chip FP8 at 4.6 PFLOPS is already 2.7×, of which the bit-width halving contributes about 2× and the N3-to-N2 process gain about 1.3 to 1.4×. Most of the 3× therefore comes from precision bit-width plus process node, and removing the VPU stall through native quantization is an increment on top of those two, bringing sustained throughput closer to the nominal peak rather than accounting for the full 3×. Alongside this, 8t widens the VPU so the VPU cycles freed by the MXU overlap vector operators (quantization, softmax, LayerNorm, stochastic rounding) with the GEMM in time, removing the bubbles at transformer block boundaries. This follows the same logic as NVIDIA Hopper introducing TMA to free the SM.

8i also has a native FP4 path, and its peak FP4 compute (10.1 PFLOPS/chip) is only 20% below 8t's (12.6 PFLOPS/chip). That 20% gap is not a trimmed compute unit but a reallocation of the silicon-area budget. The HBM configurations of the two same-generation chips make the trade-off clear: 8t has 216 GB of HBM3e at 6.5 TB/s, while 8i goes up to 288 GB at 8.6 TB/s, giving the inference chip higher bandwidth than the training chip. Google put the silicon area that would have extended the MXU array or widened the VPU into the CAE chiplet and a 384 MB on-chip SRAM array, because training demands compute more than inference does, and inference needs data reuse more than training does.

## Memory Hierarchy: Opposite Bytes/FLOP Ratios

The two chips take opposite directions in the memory subsystem. Training and inference differ by an order of magnitude in bytes/FLOP: training's forward and backward passes keep the MXU saturated, so the bottleneck is FLOPS; inference decode generates only a few KB per token yet must read the entire KV cache repeatedly, so the bottleneck is memory access.

8t slows the HBM down. Bandwidth drops from Ironwood's 7.37 TB/s to 6.5 TB/s (−12%), capacity rises from 192 GB to 216 GB (+12.5%), and on-chip SRAM holds flat at 128 MB. Google deliberately downclocks HBM3e to lower cost: FP4 halves the bytes per parameter, fusing the scale inside the MXU keeps more of the working set in SRAM, and the nominal HBM bandwidth no longer needs to keep pace. NVIDIA Rubin reaches 22 TB/s HBM4 in the same period, and Google stays on downclocked HBM3e. Training chips are bought at thousand- and ten-thousand-chip scale, so a dollar saved per chip is amplified by volume.

8i adds in the opposite direction. HBM capacity goes to 288 GB (+50%), bandwidth to 8.6 TB/s (+16%), and on-chip SRAM to 384 MB (3× Ironwood).

> With 3× more on-chip SRAM over the previous generation, TPU 8i can host a larger KV Cache entirely on silicon, significantly reducing the idle time of the cores during long-context decoding.

384 MB is enough for the per-token KV slice of a 70B-class dense model, or the active slice of a single activated expert in an MoE, to reside entirely on die, removing one HBM round trip per decode token. Bytes/FLOP rises from 8t's 0.52 to 8i's 0.85. HBM bandwidth is also higher than 8t's, because SRAM holds only the KV-cache hot region; the cold part still schedules from HBM, and agent-scenario context switching hits that path repeatedly.

## Interconnect: Same Physical Bandwidth, Different Topology

Google is decoupling functions. **SparseCore**, the unit that handles sparse data movement, remains indispensable on the training side (8t): it handles sparse-parameter gradient updates and also takes on part of the data cleaning and feature transformation. On the inference side (8i), SparseCore occupies visible chip area, so 8i replaces sparse core with a Collectives Acceleration Engine to free up die area for the CAE and a larger SRAM.

This trades memory-access-optimization resources for communication-optimization resources. The Cloud TPU product page calls it SC-CAE (SparseCore-Collectives Acceleration Engine), which suggests **CAE** inherits SparseCore's dataflow engine in its microarchitectural DNA but turns entirely toward collective communication rather than embedding. Each reduction step of auto-regressive decode no longer blocks the TensorCore's MXU, but commits into the CAE's reduction-tree FIFO to be pipelined. This change also targets MoE inference. The MoE bottleneck is not local memory access but the large volume of all-to-all communication produced by expert routing. (The Silicon Valley 101 interview with the TPU architect covers this.)

Both chips have an ICI scale-up bandwidth of 19.2 Tb/s bidirectional, 2× Ironwood. This baseline is needed by both training and inference: MoE routing doubles all-to-all traffic, and whether synchronizing expert gradients in training or forwarding tokens across chips in inference, scale-up bandwidth becomes a bottleneck if it is not raised.

The same 19.2 Tb/s hangs on different topologies. 8t keeps Ironwood's **3D torus**, 6 ICI links per chip to nearest neighbors, with a superpod at 9,600 chips. A 3D torus's bisection bandwidth grows slowly with node count and its path length is long at scale, but training does not care about single-hop latency: pretraining is synchronous mini-batch iteration, each step reducing gradients once with an all-reduce, and long paths are covered by pipelining and asynchronous prefetch. The balance a 3D torus strikes between links per chip and total wiring length is the sweet spot for large-scale synchronous training.

<figure class="tf-fig">
  <div class="tf-card">
    <svg class="tf-svg" viewBox="0 0 660 300" role="img" aria-label="3D torus topology versus Boardfly hierarchical fat-tree topology">
      <line class="divider" x1="330" y1="36" x2="330" y2="272" />
      <text class="ptitle" x="24" y="24">3D torus (8t)</text>
      <text class="ptitle" x="354" y="24">Boardfly fat-tree (8i)</text>
      <g>
        <line class="lk" x1="92" y1="72" x2="162" y2="72" /><line class="lk" x1="162" y1="72" x2="232" y2="72" />
        <line class="lk" x1="92" y1="132" x2="162" y2="132" /><line class="lk" x1="162" y1="132" x2="232" y2="132" />
        <line class="lk" x1="92" y1="72" x2="92" y2="132" /><line class="lk" x1="162" y1="72" x2="162" y2="132" /><line class="lk" x1="232" y1="72" x2="232" y2="132" />
        <circle class="nodeb" cx="92" cy="72" r="8" /><circle class="nodeb" cx="162" cy="72" r="8" /><circle class="nodeb" cx="232" cy="72" r="8" />
        <circle class="nodeb" cx="92" cy="132" r="8" /><circle class="nodeb" cx="162" cy="132" r="8" /><circle class="nodeb" cx="232" cy="132" r="8" />
        <line class="lk" x1="70" y1="94" x2="92" y2="72" /><line class="lk" x1="210" y1="94" x2="232" y2="72" />
        <line class="lk" x1="70" y1="154" x2="92" y2="132" /><line class="lk" x1="210" y1="154" x2="232" y2="132" />
        <line class="lk" x1="70" y1="94" x2="140" y2="94" /><line class="lk" x1="140" y1="94" x2="210" y2="94" />
        <line class="lk" x1="70" y1="154" x2="140" y2="154" /><line class="lk" x1="140" y1="154" x2="210" y2="154" />
        <line class="lk" x1="70" y1="94" x2="70" y2="154" /><line class="lk" x1="140" y1="94" x2="140" y2="154" /><line class="lk" x1="210" y1="94" x2="210" y2="154" />
        <path class="lkw" d="M70,94 C40,94 40,154 70,154" /><path class="lkw" d="M210,94 C240,94 240,154 210,154" />
        <circle class="node" cx="70" cy="94" r="8" /><circle class="node" cx="140" cy="94" r="8" /><circle class="node" cx="210" cy="94" r="8" />
        <circle class="node" cx="70" cy="154" r="8" /><circle class="node" cx="140" cy="154" r="8" /><circle class="node" cx="210" cy="154" r="8" />
        <text class="lblm" x="150" y="204">6 links / chip · up to 9,600 chips</text>
        <text class="lblm" x="150" y="220">long cross-pod path (&gt;10 hops)</text>
      </g>
      <g>
        <circle class="node" cx="422" cy="70" r="9" /><circle class="node" cx="522" cy="70" r="9" />
        <circle class="node" cx="386" cy="130" r="8" /><circle class="node" cx="452" cy="130" r="8" />
        <circle class="node" cx="500" cy="130" r="8" /><circle class="node" cx="566" cy="130" r="8" />
        <line class="lkf" x1="422" y1="70" x2="386" y2="130" /><line class="lkf" x1="422" y1="70" x2="452" y2="130" />
        <line class="lkf" x1="422" y1="70" x2="500" y2="130" /><line class="lkf" x1="422" y1="70" x2="566" y2="130" />
        <line class="lkf" x1="522" y1="70" x2="386" y2="130" /><line class="lkf" x1="522" y1="70" x2="452" y2="130" />
        <line class="lkf" x1="522" y1="70" x2="500" y2="130" /><line class="lkf" x1="522" y1="70" x2="566" y2="130" />
        <rect class="node" x="360" y="182" width="16" height="16" rx="3" /><rect class="node" x="396" y="182" width="16" height="16" rx="3" />
        <rect class="node" x="426" y="182" width="16" height="16" rx="3" /><rect class="node" x="462" y="182" width="16" height="16" rx="3" />
        <rect class="node" x="474" y="182" width="16" height="16" rx="3" /><rect class="node" x="510" y="182" width="16" height="16" rx="3" />
        <rect class="node" x="540" y="182" width="16" height="16" rx="3" /><rect class="node" x="576" y="182" width="16" height="16" rx="3" />
        <line class="lk" x1="386" y1="130" x2="368" y2="182" /><line class="lk" x1="386" y1="130" x2="404" y2="182" />
        <line class="lk" x1="452" y1="130" x2="434" y2="182" /><line class="lk" x1="452" y1="130" x2="470" y2="182" />
        <line class="lk" x1="500" y1="130" x2="482" y2="182" /><line class="lk" x1="500" y1="130" x2="518" y2="182" />
        <line class="lk" x1="566" y1="130" x2="548" y2="182" /><line class="lk" x1="566" y1="130" x2="584" y2="182" />
        <text class="lblm" x="472" y="222">hierarchical · network diameter more than halved</text>
      </g>
    </svg>
  </div>
  <figcaption class="tf-cap">Figure 3. 8t's 3D torus (six nearest-neighbor links per chip, long cross-pod paths) against 8i's Boardfly hierarchical fat-tree, whose maximum network diameter is said to be more than halved.</figcaption>
</figure>

8i switches topology entirely. Google calls it **Boardfly**, a hierarchical fat-tree structure (speculated), with a maximum network diameter said to be more than halved versus Ironwood. Inference's latency sensitivity differs from training's. For MoE structures, each generated token in an agent scenario can trigger expert routing, cross-chip KV scheduling, and even collectives between multiple agents, and every hop counts directly into user-facing latency. Crossing a 3D torus from one end of a pod to the other takes more than ten hops, a path length that batching absorbs in training but that shows up directly as response time on the decode path. Shortening the maximum network diameter is a direct trade for tail latency. 8i also introduces the CAE chiplet to offload frequent collective operations such as all-reduce and all-gather from the TensorCore. By Google's disclosure, the on-chip latency of a single collective drops to one-fifth, a figure that matters only for inference's high-frequency communication.

8t carries two more datapath features specific to training. **TPUDirect RDMA** lets HBM and the NIC transfer directly, bypassing the host CPU and DRAM. Training does continuous cross-node gradient all-reduce along the data-parallel dimension, where the host hop is a fixed overhead, and cutting it lowers communication latency and CPU occupancy together. **TPUDirect Storage** connects HBM directly to a Managed Lustre storage cluster (10 TB/s aggregate bandwidth, 80 PB capacity), so data goes from storage to HBM without CPU relay, about 10× faster than storage access in the Ironwood era. Frontier-model pretraining datasets already reach the hundreds-of-PB scale, and multimodal training mixes in images, video, and audio, so a data loader that cannot keep up stalls the whole training loop. This path keeps the MXU from starving on storage during ingest.

Neither feature appears on 8i. In inference, the data is model weights plus KV cache: weights load once and the KV cache resides on die, so there is no continuous large-dataset ingest from storage or cross-node gradient synchronization, and the datapath is much simpler.

## Process, Packaging, and Host

Both chips are reported as TSMC N2-class process, but this label is unconfirmed and comes mainly from The Next Platform's inference. N2's actual mass-production window is disputed, with several outlets judging that N2 reaches large-scale capacity only in the second half of 2027. The most reasonable reading is a two-step path: the 8t/8i that reach GA in the second half of 2026 may use an early N3P version, and the 2027 refresh switches to N2, corresponding to the target node for Anthropic's 3.5 GW deployment. Meta's MTIA is said to be the first AI chip in N2 mass production, and this timeline suggests TPU v8's actual ramp on N2 may run later than Google's official "later this year" phrasing implies. This process-cadence split has precedent in earlier TPU generations: ramp volume first on an early version of a mature node, then double capacity on the true next-generation node, a routine risk-reduction move for hyperscale buyers.

At the packaging level the two chips take a clear chiplet division of labor. 8t is a multi-die package of two compute dies plus one IO die plus several auxiliary chiplets, with compute area prioritized and all functions packed into the compute die to keep training workloads tightly coupled. 8i's key difference is one compute die plus one independent CAE chiplet die, separating the CAE out of the compute die into its own chiplet. This captures both yield and reuse. As a latency-sensitive module the CAE needs separate timing optimization, and as a separate die it is no longer constrained by the compute die's digital logic. The CAE module also has room for reuse on future inference chips (v8e, inference TPUs designed by a third foundry partner), and chiplet separation makes it easier to iterate on different process nodes.

Both chips drop the x86 host for the first time and switch entirely to Google's own **Axion** ARM CPU (based on Neoverse cores; whether V2 or N3, Google has not specified). The switch has both cost and customization motives: x86 hosts' licensing cost, power budget, and coupling density with Google's own IO modules are all less controllable than an in-house Arm design.

8i takes one more step on the host side, doubling the number of CPU hosts per server and enabling **NUMA** isolation, binding each TPU's data-preprocessing traffic to its local NUMA domain to avoid the cross-socket DRAM access jitter common on x86 hosts in the Ironwood era. Google's official statement is "eliminate the host bottleneck caused by data preparation latency." This optimization matters most for inference: in agentic scenarios, requests are high-concurrency and low-latency, and each request's data preprocessing (tokenization, prompt assembly, sampling-parameter parsing) is short but path-sensitive, so cross-socket DRAM jitter lands directly on user-facing latency. The 8t on the training side does not need this layer, because forward-backward iteration is a batch-level long task, host jitter is absorbed by the batch, and one host serving multiple TPUs does not become a bottleneck.

The 8t/8i split runs from the compute unit (native FP4 plus fused scale), through the memory subsystem (HBM downclock versus 3× SRAM) and interconnect topology (3D torus versus Boardfly), to packaging and the host side (tightly coupled multi-die versus independent CAE chiplet plus dual-host NUMA isolation). The two same-generation chips share a large set of IP blocks, but every layer from silicon to system takes its trade-off in the direction set by the training-inference workload difference. That is the engineering meaning of cutting one generation's design budget into two SKUs.
