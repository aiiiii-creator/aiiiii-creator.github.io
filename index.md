---
layout: home
---

<section class="hero">
  <p class="eyebrow">CHENYANG AI</p>
  <p class="lead">
    PhD student at the Institute for Computing Systems Architecture (ICSA),
    School of Informatics, University of Edinburgh. Supervised by
    Prof. Nigel Topham. Expected graduation 2028.
  </p>
  <div class="meta-row">
    <span class="meta-pill">ICSA · University of Edinburgh</span>
    <span class="meta-pill">Edinburgh, UK</span>
    <a class="meta-pill" href="mailto:C.Ai-4@sms.ed.ac.uk">C.Ai-4@sms.ed.ac.uk</a>
  </div>
  <div class="hero-actions">
    <a class="button button-primary" href="{{ '/Chenyang_Ai_FlowCV_Resume_2026-04-11.pdf' | relative_url }}">Download CV</a>
    <a class="button" href="{{ '/projects/' | relative_url }}">Selected Projects</a>
    <a class="button" href="{{ '/publications/' | relative_url }}">Publications</a>
  </div>
</section>

<section class="home-section">
  <h2>News</h2>
  <div class="timeline">
    <div class="timeline-item">
      <p class="timeline-date">Apr 2026</p>
      <h3>SNAKE preprint on arXiv</h3>
      <p>Posted <em>Rethinking Compute Substrates for 3D-Stacked Near-Memory LLM Decoding</em> on arXiv (<a href="https://arxiv.org/abs/2604.04253">2604.04253</a>) and submitted to MICRO 2026.</p>
    </div>
    <div class="timeline-item">
      <p class="timeline-date">2026</p>
      <h3>Two papers under review</h3>
      <p>SNAKE at MICRO 2026; GPNPU at ASPLOS 2027.</p>
    </div>
    <div class="timeline-item">
      <p class="timeline-date">Sep 2025</p>
      <h3>Started PhD at the University of Edinburgh</h3>
      <p>Joined ICSA, School of Informatics, working with Prof. Nigel Topham on accelerator architecture for LLM inference.</p>
    </div>
    <div class="timeline-item">
      <p class="timeline-date">Jun 2025</p>
      <h3>Completed M.Sc. at Peking University</h3>
      <p>Master's in Integrated Circuit Science and Engineering, with research on multi-precision tensor accelerators (now the GPNPU submission).</p>
    </div>
  </div>
</section>

<section class="home-section">
  <h2>Research Focus</h2>
  <div class="card-grid">
    <article class="info-card">
      <h3>3D-Stacked Near-Memory Processing</h3>
      <p>Compute substrates and scheduling frameworks for LLM decoding on 3D-stacked NMP, co-designing reconfigurable systolic arrays with multi-core orchestration.</p>
    </article>
    <article class="info-card">
      <h3>General-Purpose Tensor Accelerators</h3>
      <p>Multi-precision reconfigurable arrays and MLIR-based compilation flows that generalize NPUs toward cross-domain tensor computing.</p>
    </article>
    <article class="info-card">
      <h3>LLM-Driven Chip Generation</h3>
      <p>How large language models can drive end-to-end accelerator generation and navigate the hardware design space under functional and non-functional constraints.</p>
    </article>
  </div>
</section>

<section class="home-section">
  <h2>Selected Publications</h2>
  <div class="publication-list">
    <article class="publication-item">
      <p class="project-tag">MICRO 2026 · Under Review · arXiv</p>
      <h3>Rethinking Compute Substrates for 3D-Stacked Near-Memory LLM Decoding: Microarchitecture–Scheduling Co-Design</h3>
      <p>Chenyang Ai, Yixing Zhang, Haoran Wu, Yudong Pan, Lechuan Zhao, Wenhui OU</p>
      <p>2.90× speedup and 2.40× higher energy efficiency over state-of-the-art MAC-Tree-based 3D NMP designs across dense and MoE LLMs.</p>
      <p><a href="https://arxiv.org/abs/2604.04253">arXiv:2604.04253</a></p>
    </article>
    <article class="publication-item">
      <p class="project-tag">ASPLOS 2027 · Under Review</p>
      <h3>GPNPU: A General-Purpose NPU Architecture for Multi-Precision and Cross-Domain Tensor Operators</h3>
      <p>Chenyang Ai, et al.</p>
      <p>6.45× / 3.39× / 25.83× speedup over VPU / GPGPU / CGRA baselines, combining a multi-precision reconfigurable array with an MLIR-based compilation flow.</p>
    </article>
  </div>
  <p style="margin-top: 1.5rem;"><a class="inline-link" href="{{ '/publications/' | relative_url }}">See all publications →</a></p>
</section>
