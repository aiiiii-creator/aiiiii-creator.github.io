---
layout: home
---

<section class="hero">
  <p class="eyebrow">CHENYANG AI</p>
  <h1>ML accelerator architecture, 3D-stacked near-memory processing, and approximate hardware design.</h1>
  <p class="lead">
    I am a PhD student at the Institute for Computing Systems Architecture (ICSA),
    School of Informatics, University of Edinburgh, supervised by Dr. Jianyi Cheng,
    with Prof. Nigel Topham (Edinburgh) and Prof. George A. Constantinides (Imperial College London)
    as second supervisors. My research spans compute substrates for LLM inference on 3D-stacked NMP,
    e-graph-based approximate hardware refinement, and general-purpose tensor accelerator design.
  </p>
  <div class="meta-row">
    <span class="meta-pill">PhD Student</span>
    <span class="meta-pill">ICSA, School of Informatics</span>
    <span class="meta-pill">University of Edinburgh</span>
    <a class="meta-pill" href="mailto:C.Ai-4@sms.ed.ac.uk">C.Ai-4@sms.ed.ac.uk</a>
  </div>
  <div class="hero-actions">
    <a class="button button-primary" href="{{ '/Chenyang_Ai_FlowCV_Resume_2026-04-11.pdf' | relative_url }}">Download CV</a>
    <a class="button" href="{{ '/projects/' | relative_url }}">Selected Projects</a>
    <a class="button" href="{{ '/publications/' | relative_url }}">Publications</a>
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
      <h3>Approximate Hardware Refinement</h3>
      <p>E-graph rewriting combined with interval analysis and Lipschitz-based error propagation to explore accuracy, area, and energy trade-offs in datapath design.</p>
    </article>
    <article class="info-card">
      <h3>General-Purpose Tensor Accelerators</h3>
      <p>Multi-precision reconfigurable arrays and MLIR-based compilation flows that generalize NPUs toward cross-domain tensor computing.</p>
    </article>
    <article class="info-card">
      <h3>LLM-Driven Chip Generation</h3>
      <p>Exploring how large language models can drive end-to-end accelerator generation and navigate the hardware design space under functional and non-functional constraints.</p>
    </article>
  </div>
</section>

<section class="home-section split-section">
  <div>
    <h2>Current Work</h2>
    <div class="timeline">
      <div class="timeline-item">
        <p class="timeline-date">2025 - Present</p>
        <h3>SNAKE: Reconfigurable Compute Substrate for 3D-Stacked Near-Memory LLM Decoding</h3>
        <p>A low-area reconfigurable systolic array and multi-core scheduling framework tailored for LLM decode on HBM3-class 3D-stacked NMP. Submitted to MICRO 2026.</p>
      </div>
      <div class="timeline-item">
        <p class="timeline-date">2025 - Present</p>
        <h3>Approximate E-graph Rewriting for Hardware Refinement</h3>
        <p>Combines equality saturation with approximate computing, using interval analysis and Lipschitz-based error propagation for sound datapath refinement. Submitted to ASPLOS 2027.</p>
      </div>
      <div class="timeline-item">
        <p class="timeline-date">2025 - Present</p>
        <h3>LLM-Based Automatic Chip Generation</h3>
        <p>Investigating how LLMs can drive end-to-end generation of accelerator RTL and navigate the architectural design space.</p>
      </div>
      <div class="timeline-item">
        <p class="timeline-date">2024 - 2025</p>
        <h3>GPNPU: General-Purpose NPU for Multi-Precision and Cross-Domain Tensor Operators</h3>
        <p>A reconfigurable multi-precision array plus MLIR-based compilation flow that lifts cross-domain tensor operators into NPU-compatible execution. Submitted to ASPLOS 2027.</p>
      </div>
    </div>
  </div>
  <div class="highlight-panel">
    <p class="highlight-label">HIGHLIGHTS</p>
    <h3>Selected snapshots</h3>
    <ul class="status-list">
      <li>PhD student at the University of Edinburgh (ICSA), expected graduation in 2028.</li>
      <li>Three papers under review at MICRO 2026 and ASPLOS 2027.</li>
      <li>GPNPU achieves 6.45× / 3.39× / 25.83× speedup over VPU / GPGPU / CGRA baselines.</li>
      <li>SNAKE delivers 2.90× speedup and 2.40× higher energy efficiency over state-of-the-art 3D NMP designs.</li>
      <li>National First Prize, China College Integrated Circuit Competition (2021).</li>
    </ul>
    <a class="inline-link" href="{{ '/about/' | relative_url }}">Full profile</a>
  </div>
</section>

<section class="home-section">
  <h2>Selected Publications</h2>
  <div class="publication-list">
    <article class="publication-item">
      <p class="project-tag">MICRO 2026 · Under Review · arXiv</p>
      <h3>Rethinking Compute Substrates for 3D-Stacked Near-Memory LLM Decoding: Microarchitecture–Scheduling Co-Design</h3>
      <p>Chenyang Ai, Yixing Zhang, Haoran Wu, Yudong Pan, Lechuan Zhao, Wenhui OU</p>
      <p>A reconfigurable systolic array tailored for LLM decode on 3D-stacked NMP, together with a multi-core scheduling framework that aligns systolic dataflows with spatial and spatio-temporal partitioning.</p>
      <p><a href="https://arxiv.org/abs/2604.04253">arXiv:2604.04253</a></p>
    </article>
    <article class="publication-item">
      <p class="project-tag">ASPLOS 2027 · Under Review</p>
      <h3>GPNPU: A General-Purpose Neural Processing Unit Architecture for Multi-Precision and Cross-Domain Tensor Operators</h3>
      <p>Chenyang Ai, et al.</p>
      <p>A multi-precision reconfigurable array and MLIR-based compiler that lowers cross-domain tensor operators onto NPUs.</p>
    </article>
    <article class="publication-item">
      <p class="project-tag">ASPLOS 2027 · Under Review</p>
      <h3>Approximate E-graph Rewriting for Hardware Refinement</h3>
      <p>Chenyang Ai, Jianyi Cheng, et al.</p>
      <p>E-graph-based approximate datapath synthesis with interval analysis and Lipschitz-bounded error propagation.</p>
    </article>
  </div>
</section>
