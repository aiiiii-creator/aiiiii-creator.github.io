---
layout: home
---

<section class="hero">
  <p class="eyebrow">CHENYANG AI</p>
  <h1>ML accelerator architecture, compiler systems, and hardware/software co-design.</h1>
  <p class="lead">
    I am a master's student in Integrated Circuit Science and Engineering at Peking University.
    My recent work spans vector and systolic architectures, tensor compiler workflows based on MLIR,
    and analytical frameworks for understanding the generality of ML accelerators.
  </p>
  <div class="meta-row">
    <span class="meta-pill">Peking University</span>
    <span class="meta-pill">Beijing, China</span>
    <a class="meta-pill" href="mailto:chenyang_ai@stu.pku.edu.cn">chenyang_ai@stu.pku.edu.cn</a>
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
      <h3>NN Hardware/Software Co-design</h3>
      <p>Architecture exploration across model, compiler, and hardware boundaries for efficient ML execution.</p>
    </article>
    <article class="info-card">
      <h3>Tensor Compiler and MLIR</h3>
      <p>Compilation flows that lower tensor operators into executable kernels and map them onto heterogeneous accelerators.</p>
    </article>
    <article class="info-card">
      <h3>VPU, GPU, NPU, CGRA</h3>
      <p>Microarchitectural analysis of programmable and spatial accelerators for diverse tensor operators.</p>
    </article>
    <article class="info-card">
      <h3>Chiplet Design Space Exploration</h3>
      <p>Cross-layer studies on utilization, scalability, and the system-level tradeoffs of emerging accelerator organizations.</p>
    </article>
  </div>
</section>

<section class="home-section split-section">
  <div>
    <h2>Current Work</h2>
    <div class="timeline">
      <div class="timeline-item">
        <p class="timeline-date">Mar 2024 - Present</p>
        <h3>Tensor Train Full-Process Accelerator for LLMs</h3>
        <p>Exploring TT decomposition and dedicated architectures that accelerate both factorization and vector-matrix operations during decoding.</p>
      </div>
      <div class="timeline-item">
        <p class="timeline-date">Apr 2024 - Present</p>
        <h3>Analysis Framework for the Generality of ML Accelerators</h3>
        <p>Building a tensor-algebra-based framework to search for effective hardware configurations across arbitrary operator ranges.</p>
      </div>
      <div class="timeline-item">
        <p class="timeline-date">Jan 2024 - Jun 2024</p>
        <h3>Heterogeneous SoC Compiler Based on MLIR</h3>
        <p>Mapped tensor contractions to matrix multiplication and built an MLIR-based toolchain for scheduling kernels across accelerators.</p>
      </div>
    </div>
  </div>
  <div class="highlight-panel">
    <p class="highlight-label">HIGHLIGHTS</p>
    <h3>Selected snapshots</h3>
    <ul class="status-list">
      <li>Master's student at Peking University since September 2022.</li>
      <li>Two DAC 2025 papers currently under review.</li>
      <li>CCF CHIP'24 paper invited for journal extension.</li>
      <li>Ranked 1/35 in undergraduate major with GPA 3.79/4.0.</li>
      <li>National competition awards in IC design, speech, and sign-language acceleration.</li>
    </ul>
    <a class="inline-link" href="{{ '/about/' | relative_url }}">Full profile</a>
  </div>
</section>

<section class="home-section">
  <h2>Selected Publications</h2>
  <div class="publication-list">
    <article class="publication-item">
      <p class="project-tag">UNDER REVIEW</p>
      <h3>MPTVPU: A RISC-V VPU Could Solve the Problems of Precision and Tensor</h3>
      <p>Chenyang Ai, Lechuan Zhao, Zhijie Huang, Cangyuan Li, Xinan Wang, Ying Wang</p>
      <p>DAC 2025 submission. <a href="https://arxiv.org/abs/2405.02196">arXiv</a></p>
    </article>
    <article class="publication-item">
      <p class="project-tag">UNDER REVIEW</p>
      <h3>What Is Relationships between Utilization and Generality of Systolic Array: Insights and Solution</h3>
      <p>Chenyang Ai*, Lechuan Zhao*, Xinan Wang, Ying Wang</p>
      <p>DAC 2025 submission focused on utilization and generality tradeoffs in systolic arrays.</p>
    </article>
    <article class="publication-item">
      <p class="project-tag">INVITED EXTENSION</p>
      <h3>HVSA: A Deeply Hybrid Vector Systolic Architecture with Dynamically Reconfigurable Dataflow</h3>
      <p>Chenyang Ai*, Lechuan Zhao*, Xinan Wang, Ying Wang</p>
      <p>Presented at CCF CHIP'24 in Chinese and invited for a further submission to Journal of Computer Science and Technology.</p>
    </article>
  </div>
</section>
