---
layout: page
title: Projects
permalink: /projects/
---

## PhD Research · University of Edinburgh (2025 – Present)

<div class="project-grid">
  <article class="project-card">
    <p class="project-tag">MICRO 2026 · Under Review</p>
    <h2>SNAKE: Reconfigurable Compute Substrate for 3D-Stacked Near-Memory LLM Decoding</h2>
    <p><strong>Role:</strong> Lead author — microarchitecture and scheduling co-design.</p>
    <p>We rethink the compute microarchitecture of 3D-stacked near-memory processing and replace prior MAC-Tree-based compute units with a reconfigurable systolic array tailored for LLM decode. The array is tightly integrated with the vector core to provide both systolic efficiency and vector-style flexibility under the tight logic-die area budget.</p>
    <p>We further scale SNAKE to multi-core logic dies with a scheduling framework and lightweight on-chip interconnect that aligns systolic dataflows with spatial and spatio-temporal partitioning.</p>
    <p>Compared to state-of-the-art 3D NMP baselines, SNAKE achieves <strong>4.00× compute-area efficiency</strong>, <strong>2.90× speedup</strong>, and <strong>2.40× higher energy efficiency</strong> across dense and MoE LLMs.</p>
    <p><a href="https://arxiv.org/abs/2604.04253">arXiv:2604.04253</a></p>
  </article>

  <article class="project-card">
    <p class="project-tag">ASPLOS 2027 · Under Review</p>
    <h2>Approximate E-graph Rewriting for Hardware Refinement</h2>
    <p><strong>Role:</strong> Lead author, with Dr. Jianyi Cheng.</p>
    <p>An e-graph-based framework that co-explores exact and approximate rewrites within a unified abstraction, enabling datapath refinement under configurable accuracy budgets.</p>
    <p>The framework combines interval analysis with Lipschitz-based error propagation to bound compositional approximation errors, and formulates hardware extraction as an ILP problem over area and error.</p>
    <p>Benchmarks cover Herbie numerical kernels as well as dot-product and constant-weight accumulation patterns commonly found in ML and DSP datapaths.</p>
  </article>

  <article class="project-card">
    <p class="project-tag">2025 - Present</p>
    <h2>LLM-Based Automatic Chip Generation</h2>
    <p><strong>Role:</strong> Lead researcher.</p>
    <p>Exploring how large language models can drive end-to-end accelerator generation, from architectural specification to RTL, and navigate the hardware design space under functional and non-functional constraints.</p>
  </article>
</div>

## Master's Research · Peking University (2022 – 2025)

<div class="project-grid">
  <article class="project-card">
    <p class="project-tag">ASPLOS 2027 · Under Review</p>
    <h2>GPNPU: General-Purpose NPU for Multi-Precision and Cross-Domain Tensor Operators</h2>
    <p><strong>Role:</strong> Lead author — microarchitecture, compiler, and evaluation.</p>
    <p>We identify the similarity between matrix multiplication and multi-precision multiplication, and propose a Multi-Precision Reconfigurable Array (MPRA) integrated into a VPU to form a General Tensor Accelerator (GTA).</p>
    <p>On the software side, GTA-Codegen is an MLIR-based compilation framework that extracts high-level multi-precision tensor semantics across domains and performs precision-aware array scheduling on the GTA.</p>
    <p>Compared with VPU, GPGPU, and CGRA baselines, GTA achieves <strong>6.45× / 3.39× / 25.83× speedup</strong> and <strong>7.76× / 5.35× / 8.76× memory-efficiency improvements</strong>, respectively.</p>
  </article>
</div>

## Undergraduate Research · CQUPT (2018 – 2022)

<div class="project-grid">
  <article class="project-card">
    <p class="project-tag">National 1st Prize · 2021</p>
    <h2>End-to-End FPGA Accelerators for CNN, LSTM, and Attention</h2>
    <p><strong>Role:</strong> Project leader.</p>
    <p>Implemented three FPGA-based accelerators and drove the full pipeline in real-world scenarios — from dataset collection and neural-network training to dedicated hardware design — covering CNN, LSTM, and attention models together with audio and image processing kernels.</p>
    <p>Awarded the <strong>National First Prize at the China College Integrated Circuit Innovation and Entrepreneurship Competition</strong> for an SoC that combines sign-language and speech recognition on these accelerators.</p>
  </article>
</div>
