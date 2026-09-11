---
layout: post
title: "Symbolic Methods, Heuristics, and AI Search in AI Compilers"
date: 2026-05-11
categories: [compilers]
excerpt: "Four generations of search for the same problem — find a near-optimal program transformation in a space of 10^15 while preserving semantics — and where each one actually belongs: e-graphs, learned cost models, RL and MCTS, and LLMs."
---

*Four generations of search for the same problem — find a near-optimal program transformation in a space of 10^15 while preserving semantics — and where each one actually belongs: e-graphs, learned cost models, RL and MCTS, and LLMs.*

This piece leans toward algorithms, but it's also an important line of applied thinking (it felt like being back in undergrad math-modeling competitions). It surveys the design ideas, representative work, and appropriate layer for each of four classes of search method in AI compilation: symbolic methods, represented by e-graphs and equality saturation; heuristic methods centered on cost models; AI search methods represented by RL, MCTS, and evolutionary algorithms; and the recently emerging LLM-assisted compiler optimization.

*This is a translation of a piece originally published on Zhihu on May 11, 2026.*

## 1. The core problem of AI compilation

The core problem of AI compilation is essentially the same as hardware design-space exploration, or any search problem: in a combinatorially huge space of program transformations, approach the near-optimal solution for some performance or resource target within limited time, while preserving semantic equivalence. The problem has three sets of constraints.

The first is search space versus time budget. The operator-level schedule space of one transformer block is above 10^15. An LLVM pass pipeline, even restricted to the hundred-odd commonly used passes, has a huge number of combinations. But compile time has to be bounded — online scenarios can't take days, and offline tuning doesn't want to burn thousands of GPU-hours either.

The second is correctness versus performance. Every transformation must preserve semantic equivalence, but the more aggressive the transformation (fusion, rewriting, quantization, approximation), the easier it is to get wrong.

The third is generality versus specialization. The more general a search algorithm — adapting to many hardware targets and many operators — the harder it usually is to approach the performance of hand-written libraries (cuBLAS, cuDNN).

Around these three constraints, the community has gone through roughly four generations of methods over the past decade. The first was heuristics plus templates: Halide, AutoTVM, XLA's pattern matching. The second was symbolic methods plus ILP: egg, TASO, Tensat. The third was learned search: Ansor, X-RLflow, AlphaTensor. The fourth is LLM-assisted and agentic methods: Meta's LLM Compiler, Compiler-R1.

One more note: these four generations aren't replacements but layers. Today's industrial compiler stacks (TVM, XLA, Triton, IREE) are almost all hybrids — pattern-matching heuristics at the graph level, a cost model plus evolutionary search at the kernel level, RL or an LLM plugged in for certain scenarios. So treat them as complementary tools.

## 2. Symbolic methods: e-graphs and equality saturation

Strictly speaking, an e-graph is a representation, not a search method. What it does is describe the set of "all equivalent forms of a program" with one compact data structure. Equality saturation is the accompanying construction process (filling that set using rewrite rules), and extraction is the actual search (choosing the best from the constructed structure).

### 2.1 The phase-ordering problem

An example: y = ReLU(BN(Conv(x, W)))

There are typically three rewrite rules for this pattern. Rule A is BN folding: at inference, BatchNorm can be folded into Conv's weights and bias. Rule B is Conv-BN fusion: use the hardware's FusedConvBN kernel to execute the two operators together. Rule C is Conv-ReLU fusion: a FusedConvReLU kernel with epilogue support.

Apply A first and BN disappears, B no longer matches, and you can only apply C on the plain Conv, ending with FusedConvReLU. Apply B first and you get FusedConvBN, A is no longer available; unless the hardware supports a ReLU epilogue after FusedConvBN, ReLU has to execute separately and part of the fusion opportunity is lost. If the hardware offers a three-in-one FusedConvBNReLU kernel, the optimum requires matching the whole pattern directly, and any path that applies A or B alone first destroys that opportunity.

Extend to Conv-BN-ReLU followed by Add (the residual connection), the basic ResNet block, and the interdependence among rules gets more complex. Some rules require the two branches before the Add to have specific data layouts, and the layout transform may conflict with BN folding. Once the rule set reaches a certain size, the final result of serial application becomes rapidly more sensitive to order, and the performance gap can be several times. That's the phase-ordering problem as it appears in tensor-graph rewriting.

### 2.2 E-graphs

Equality saturation offers an approach: instead of choosing rewrites serially, record all reachable equivalent expressions at once, then extract the best at the end. The data structure that carries this idea is the e-graph (equality graph).

An e-graph has two kinds of node. An e-node is an operator plus references to child e-classes. An e-class is a set of semantically equivalent e-nodes. When two e-nodes' corresponding child e-classes are all equal, they're merged into the same e-class — congruence closure.

E-graphs use equivalence classes as the basic unit, letting one graph represent exponentially many equivalent expressions at once.

The application flow thus becomes two phases. Phase one is saturation: apply rewrites repeatedly until the e-graph stops changing or a resource limit is reached. Phase two is extraction: run a cost-aware selection algorithm (greedy or ILP) over the e-graph to find the cheapest equivalent expression.

The name "equality saturation" first appeared in Tate, Stepp, Tatlock, and Lerner's POPL '09 paper, with the accompanying system Peggy. Technically complete, but the e-graph implementation of the time was inherited from 1970s theorem provers (Nelson, Oppen, Downey-Sethi-Tarjan), and under a rewrite-heavy saturation workload it was slow and hard to scale. For a decade it existed mainly as an internal component of theorem provers (Z3 and Simplify both use e-graphs for congruence closure), with little standalone compiler-optimization work. Denali, equality saturation itself, and Herbie were the few representatives; the community was tiny.

egg won a Distinguished Paper award at POPL 2021. It didn't invent a new concept; it engineered e-graphs to the point where others could pick them up and use them directly. Through e-class analysis it supports binding additional information (tensor shape, cost) to e-classes, making cost-aware rewriting and extraction possible. Much later work builds on egg.

Tensat (MLSys '21) applied equality saturation to DNN graph optimization. Its approach: reuse the graph-substitution rules TASO synthesized (both single-pattern and multi-pattern rewrites), build the e-graph with egg in place of TASO's backtracking search, and use ILP for the final graph extraction. Experiments showed Tensat covering a larger search space than TASO and reaching better performance in less time. That shows the character of the e-graph abstraction: the phase-ordering problem is structurally eliminated — all rewrites apply simultaneously, and the optimal choice is deferred to the last step.

### 2.3 Strengths and weaknesses

#### Strengths

**Phase ordering eliminated structurally.** Heuristics and AI search both face phase ordering by "searching for a better order in a larger decision space." Backtracking search enumerates orders, RL learns a long-horizon value function, MCTS balances exploration and exploitation with UCB. These are all efforts at the level of the optimization algorithm.

E-graphs aren't in that frame. Through non-destructive rewriting, all equivalent forms coexist, and the application-order dimension is eliminated outright rather than searched more cleverly. That's dimensionality reduction, not optimization. In scenarios where phase ordering is severe (rules interfere, local cost doesn't predict global performance), the e-graph's advantage is pronounced.

**Composable and extensible.** Adding a new rewrite rule to an e-graph doesn't affect the correctness of existing rules and doesn't require retraining any model. Equivalence classes only grow, never shrink.

Adding an action to an RL agent usually means retraining, because the policy network's output dimension changes and the new action may shift the state distribution. Adding a rule to a heuristic method means re-tuning weights or priorities, especially when rules conflict. This "add things without breaking old things" property makes e-graphs suited to long-lived engineering systems: a project can start with a few rules and grow the rule set as needs grow.

**Transformations are trustworthy and traceable.** Every transformation on an e-graph corresponds to one rewrite rule. Rules are local equivalence propositions that can be formally verified; congruence closure extends local equivalence to global. If the rules are correct, the result is guaranteed semantically equivalent — no verifier needed. The same mechanism leaves a complete trail: which e-nodes compose the final extracted term, which rule each e-node came from, all traceable.

Heuristic methods' correctness depends on the design of the schedule primitives. TVM's and Halide's schedule operations don't change semantics, but the property is implicit and easy to break when extending with new primitives. RL relies on environment verification: the agent proposes an action, the compiler actually executes and checks, and out-of-bounds actions fail to compile; the learned policy is a black box, and when it produces a strange schedule it's hard to tell bug from optimization. LLM output can be syntactically correct but semantically wrong, needs an external verifier before production, and the process is hard to trace.

Compilers are correctness-sensitive and debugging-intensive; the e-graph's advantage on both counts is structural.

#### Weaknesses

**Only fits discrete rule spaces.** The e-graph's foundation is "a finite set of rewrite rules" and "the discrete structure of terms." Schedule tuning faces continuous parameters (tile size any power of two, unroll factor any integer from 1 to N) and combinatorial spaces (the Cartesian product of several parameters); stuffing all of that into an e-graph is neither realistic nor useful.

At that layer, heuristics plus a learned cost model are more direct: enumerate the parameter space in a structured way, score each candidate with the cost model, choose with evolution or Bayesian optimization. RL can handle it too (actions being parameter choices), but in engineering practice evolutionary algorithms are often more stable. That's why the Ansor / MetaSchedule line remains the industrial mainstream, and e-graphs are basically absent at this layer.

**They blow up on their own.** The e-graph's compression relies on "high sharing and strong rule compositionality." Break that premise and the e-graph itself grows exponentially. Multi-pattern rewrites are the classic trigger: a rule requires two patterns to coexist in the e-graph before it applies, and newly added e-nodes trigger more pattern matches, quickly spiraling.

Heuristics and RL walk a single path through the state space, taking linear memory however deep they go. The e-graph holds the entire equivalence space in memory, and at scale it can't proceed. Tensat had to limit multi-pattern rewrites to 1–2 rounds, and the core motive of work like MCTS-guided EqSat is to control the e-graph's own expansion. This is the main reason e-graphs still struggle with "complex industrial scenarios with 100+ rules."

**Extraction is NP-hard.** After saturation there's still the problem of finding the best equivalent expression. Cost-aware extraction on a DAG with sharing is equivalent to weighted DAG covering, NP-hard in general. Greedy bottom-up extraction is linear but can get stuck in local optima. ILP extraction gives the optimum but only handles a few thousand e-classes.

Heuristics and the other AI methods produce "one solution" to begin with, with no question of global optimality. The e-graph turns finding the optimum into an explicit step, exposing this NP-hard problem. MCTS-guided extraction is a newer compromise, still at the research stage.

**Rules have to be written by hand.** The e-graph's capability ceiling is set entirely by the rule set. Rules are either mathematical identities (associativity, distributivity) or domain knowledge (fusion patterns, hardware-specific equivalences), and the latter depends entirely on experts. (This one is so annoying — lately I've become a hand-written-rewrite immortal.)

RL and LLMs have a structural advantage here: they can learn from data and don't need explicit rules. When AlphaTensor searched the space of matrix-multiplication decompositions, nobody told it "what transformations to try"; the reward signal guided it to discover the rules. LLM Compiler read 546 billion tokens of LLVM code and implicitly learned a large number of optimization patterns. Rule-synthesis work like Ruler is letting e-graphs "discover rules automatically" too, but maturity and coverage still lag RL and LLMs.

**High integration cost.** Integrating an e-graph into an existing compiler means translating the IR into e-node representation, defining a cost function, writing the rule set, configuring saturation (iteration count, memory cap, rule priority), and finally writing the extracted term back into the original IR. None of the steps is hard, but together they're several person-months of engineering.

Integrating a heuristic method is writing a search loop; integrating RL is wrapping a Gym environment — both lighter. That's why e-graphs show up more in newly written tools (Tensat, Cranelift's mid-end) and spread slowly in the mainline of large existing codebases like LLVM and TVM. They suit adoption at a new project's start, not a mid-course switch.

## 3. Searching the schedule space: heuristics and learned cost models

### 3.1 The schedule abstraction

The key modeling at the tensor-program level comes from Halide (PLDI '13) and TVM (OSDI '18), which together established the compute/schedule separation. Compute describes what to compute — GEMM is a few lines of loops with multiply-accumulate; schedule describes how — tile size, loop order, whether to vectorize, whether to pipeline, which cache level to keep intermediates in. The same compute with different schedules can differ in performance by tens of times.

This separation restated "search for a high-performance kernel" as "search for a good schedule." Why is that step key? Because it defines the entire tensor-program-level search space in a structured way: every schedule dimension is a discrete parameter, and the legal combinations of parameters are guaranteed not to break semantics by the schedule primitives. All later automation builds on this modeling; at bottom it's all the same thing — find a good solution in the space spanned by the schedule primitives.

But once the space is explicit, it turns out to be enormous. A GEMM's schedule space is above 10^15. It's worth being precise here: the "hand-written kernel" route was never abandoned by industry — cuBLAS, cuDNN, and oneDNN remain the performance baseline, and they are highly tuned hand-made products. What was abandoned was a different paradigm: having an expert fill in a fixed schedule for every operator and every shape. That exhaustive manual programming doesn't scale. The core problem later methods solve is how to search this space automatically at all.

### 3.2 AutoTVM: using a cost model to free the search from the hardware

What stalls the search is that hardware measurement is too slow. Measuring one candidate kernel means running real hardware, warming up, taking the median of several runs — hundreds of milliseconds to seconds each. With a large space, the time budget evaporates instantly.

AutoTVM (NeurIPS '18) introduces a learned cost model to replace most hardware measurements. The developer first writes a schedule template with tunable parameters (tile_size, unroll_factor, and so on); the system runs a few samples on hardware to train a GBM or a small neural network as a cost model, then uses the cost model to screen the top-k from many candidates, sending only the top-k to hardware for measurement. The measurements feed back to keep training the cost model, iterating until the budget is spent. The exploration strategy supports random, grid, simulated annealing, GA, and others; simulated annealing is the default recommendation.

This "few real measurements, many proxy evaluations" paradigm pushed the cost of one evaluation from hundreds of milliseconds to a few milliseconds, and the search scale grew by orders of magnitude. It's still the skeleton of most auto-tuners.

But AutoTVM left an obvious limitation: the templates still had to be written by hand. Every new operator class (Conv2D, GEMM, Pooling, LayerNorm) needs an expert to write a template, and how the template is written sets the boundary of the search space. Write conservatively and you lose optimization opportunities, and the expert can't anticipate every valuable combination anyway.

### 3.3 Ansor: automating the template step too

Ansor (OSDI '20, called Auto-Scheduler in TVM) aims to remove the manual template. It uses hierarchical sketches to generate a schedule skeleton directly from the compute definition — the skeleton describes structural choices (how many tile levels, which loops parallelize, at which level to do cache reads), and the concrete parameters inside the skeleton are filled by evolutionary search. A cost model still evaluates, plus a task scheduler that dynamically allocates the search budget among subgraphs, spending compute first on the subgraphs that most affect end-to-end latency.

Ansor chose evolutionary search rather than AutoTVM's exploration strategies not because the algorithm is inherently superior but because the space it has to search is an order of magnitude larger — no longer bounded by hand-written templates. Evolutionary algorithms take long strides in a large space through mutation and crossover, and are relatively robust to cost-model noise; the fit is good.

In the data, Ansor achieved up to 3.8×, 2.6×, and 1.7× speedups on Intel CPU, ARM CPU, and NVIDIA GPU respectively. The paper has one observation worth singling out: the best schedules Ansor found lay outside the search spaces of existing search-based methods. That is, progress wasn't searching the same space faster; the space itself grew, covering optimization combinations hand-written templates could never reach. That insight recurs through this chapter: **the expressiveness of the search space is the true ceiling for this class of method; the cleverness of the search algorithm is secondary.**

The line has moved further since Ansor. TVM introduced MetaSchedule, turning the schedule into a programming language so sketches and search strategies are expressed in one language, lowering the bar for writing new schedule systems further; FamilySeer noticed that different subgraphs are often structurally similar and let them share cost models to improve search efficiency. The paradigm's direction is clear: humans write less and less, and automation covers deeper and deeper.

### 3.4 The cost model is the paradigm's Achilles' heel

The whole method's effectiveness is staked on the cost model. If the cost model is inaccurate, evolutionary search burns budget in the wrong direction; if it doesn't generalize, every new hardware class requires retraining. Around this core, a few engineering approaches are common: encode the schedule as a fixed-length feature vector (loop counts, memory access pattern, arithmetic intensity) and train XGBoost or LightGBM; use a TreeLSTM or GNN directly on the schedule's tree structure; serialize the schedule and encode it with a Transformer (the One-Shot Tuner family); or a hybrid analytical model plus ML residual correction.

In practice, the truly unavoidable problems are connected. First, **real measurement is noisy** — CPU background processes and GPU frequency adjustment make the same kernel vary by several percent across runs, so you need the median of several runs to stabilize. Second, the cost model **drifts** during search: early low-quality samples and later high-quality candidates have different distributions, and without continuous feedback training the cost model deviates more the longer it searches. Together these make **cross-hardware transfer** hard — the best schedule for the same code is completely different on V100, A100, and H100, and since the cost model is sensitive to noise and distribution, it can almost only be trained per device, and the initialization dividend from pretraining is quickly consumed.

### 3.5 The performance wall: search-space expressiveness is the ceiling

The cost model's limits lead directly to a phenomenon: on mainstream operators on mainstream hardware (FP32/FP16 GEMM, Conv), the auto-tuner's best results often still trail cuBLAS and cuDNN, with the bottleneck concentrated in Tensor Core utilization. The reason is that hardware details like Tensor Core orchestration, warp scheduling, and bank conflicts hugely affect performance, but a traditional cost model's feature space can't capture them.

The evolution of later work confirms this from the other direction. Roller (OSDI '22), AMOS (ISCA '22), and TensorIR (ASPLOS '23) share one thing: they abstract the Tensor Core directly into the search space — not a smarter search algorithm, but exposing the hardware hierarchy (thread-block tile, warp tile, MMA fragment) as tunable dimensions. Once that dimension is in the search, heuristic methods can approach or even catch cuBLAS on FP16 GEMM. Triton and CUTLASS take another route, giving developers higher-level hardware abstractions so people can write the hardware details at the right granularity. Both routes reach the same conclusion: **the search's ceiling is set by the search space's expressiveness, not the search algorithm's cleverness** — the continuation of 3.3's observation.

But even with an expanded search space, one engineering reality remains: **compile time**. AutoTVM and Ansor often take hours to days to tune a whole model, and the deeper the search and the larger the space, the longer it takes. That's the core reason they get bypassed in many real settings — TorchInductor and torch.compile use lighter templates plus a little tuning, trading peak performance for acceptable compile time, and are more popular in production as a result.

Despite losing to hand-written libraries and being slow to tune, heuristics plus a learned cost model remain the first choice in research-grade production environments. The reasons are pragmatic: no dependence on a pretrained large model, no need for RL's massive exploration; relatively interpretable behavior, and a cost model you can debug when it goes wrong; decent adaptability to unseen hardware — as long as you can run measurements you can train a new cost model. TVM, XLA, and IREE all remain based on this paradigm — not because it's the strongest, but because it's the only one that passes on "good enough, controllable, maintainable" all at once.

The AI search methods and LLM methods in the rest of this piece are each stronger than heuristics on some dimension, but none passes on all three at once yet.

## 4. AI search methods: RL, MCTS, and evolutionary algorithms

Heuristic methods hand "which candidate to search" to some fixed strategy (greedy, evolutionary, simulated annealing). RL and MCTS make that strategy itself learnable. Following the previous chapter's judgment — search-space expressiveness is the ceiling — the key contribution of the RL/MCTS line isn't just smarter search but redefining "what counts as searchable": from one schedule parameter at a time to whole long decision sequences.

### 4.1 Where RL fits compiler optimization

Compiler optimization structurally fits reinforcement learning. The state is the current IR, e-graph, or schedule; the action is applying a rewrite, choosing a pass, setting a tile size; the reward is the runtime after compilation, or a proxy cost-model estimate; an episode is the full path from source program to final lowered code.

What gives RL unique value on this class of problem is **long-horizon credit assignment**. A rewrite that looks locally disadvantageous — an apparently "backward" algebraic simplification, say — may create the conditions for a more aggressive later optimization. Greedy won't pick it, since the short-term metric worsens; evolutionary algorithms may stumble into it but have no explicit mechanism to learn "accept a loss at step 2 for a gain at step 8" into the policy. RL's value function exists for exactly this.

### 4.2 X-RLflow

X-RLflow (VLDB '23) is a direct instance of this idea at the graph-rewrite level. It replaces TASO's cost-based backtracking search with a deep RL agent: a GNN encodes the current tensor graph, outputs a Q-value for each rewrite rule, and decides step by step which rewrite to apply next. Experiments show X-RLflow beating TASO on several workloads, precisely because the agent can give up short-term gains for long-term ones — structurally impossible for pure greedy and cost-based search.

### 4.3 AlphaTensor: making "which algorithm" a search target too

DeepMind's AlphaTensor (Nature, 2022) models "finding fast matrix-multiplication algorithms" as a single-player game. The state is a 3D tensor initialized to the matrix-multiplication tensor T_{n,m,p}; an action "subtracts" a rank-1 term u⊗v⊗w from the tensor; the game ends when the tensor reaches zero, and the number of rank-1 terms used is the algorithm's multiplication count; the reward is the negative step count.

That's a standard AlphaZero-style setup — a neural network plus MCTS. The result: AlphaTensor found a 47-multiplication algorithm for 4×4 matrix multiplication over GF(2), beating the 49-multiplication record Strassen's algorithm had held in that setting since 1969. It can also optimize for measured latency on specific hardware (V100, TPU), finding algorithms faster than the standard implementation.

From the AI-compilation angle, AlphaTensor's significance isn't "beating Strassen's record" as such but **pushing the boundary of search to somewhere more fundamental than the schedule** — "which algorithm implements this operator" is itself in the search space. The previous chapter took search-space expressiveness as the ceiling; this is an explicit raising of that ceiling: no longer choosing a schedule within a given algorithm's implementations, but re-choosing the algorithm. The cost is that the search space jumps from 10^15 straight to combinatorial explosion, navigable only by a combination with strong planning like MCTS plus neural networks.

### 4.4 RL at the LLVM and MLIR levels

In classical compilation, CompilerGym (CGO '22) wraps LLVM phase ordering, GCC flag selection, CUDA loop-nest generation, and other tasks as OpenAI Gym-compatible RL environments, letting RL researchers evaluate algorithms directly on compiler problems. It has become a common benchmark in the area.

More recent MLIR RL work (CGO '26) applies RL to the MLIR Linalg dialect, proposing a multi-action RL formulation (the action space as a Cartesian product of simple sub-actions) and a level-pointer trick to shrink the loop-interchange action space. The shared motive: a multi-dialect IR like MLIR gives RL more structured state and action semantics than raw LLVM — RL learns poorly in spaces with fuzzy state/action semantics, and the more regular the IR, the better the sample efficiency.

### 4.5 Evolutionary algorithms and Bayesian optimization: not all "AI search" needs RL

In engineering practice, evolutionary algorithms plus a learned cost model (the Ansor setup) are often more stable and need less tuning than pure RL. The reason is structural: evolutionary algorithms are more robust to reward noise, needing none of the numerically sensitive advantage-estimation machinery; evaluation is naturally parallel, using CPU/GPU resources better; and the failure mode is simple — "population degeneration" — far cheaper to diagnose than an RL training collapse.

Where RL genuinely pays off is **long-episode, sparse-reward, planning-heavy** tasks — phase ordering, graph-level rewriting — which apply many transformations in sequence before a final evaluation, where greedy and evolutionary approaches suffer. For short-episode, dense-reward schedule tuning, an evolutionary algorithm or Bayesian optimization is usually enough, and bolting RL on buys little for its complexity.

### 4.6 The cost of AI search

The costs are unavoidable. Training is expensive — AlphaTensor's training compute far exceeds an ordinary auto-tuner's. Generalization is limited — a policy an RL agent learned on ResNet-50 may not transfer directly to ViT. Implementation is complex — RL training involves environments, reward design, replay buffers, and stability tricks, more complex than an evolutionary search loop. Interpretability is low — when the agent produces a strange schedule, engineers can't tell whether it's a bug or genuinely good. That last point lands squarely on the "controllable" and "maintainable" items of 3.5's three-part standard.

These costs mean RL and MCTS are currently used mainly in research and in a few high-value scenarios (hand-written GEMM, very-large-model deployment, algorithm discovery), and haven't become mainstream compilers' default strategy. They are genuinely stronger in search-space expressiveness, but haven't yet passed on all three counts at once.

## 5. LLM-assisted compiler optimization

Since 2023, LLMs have entered the compiler field fairly quickly (I can say I was on one of the earliest LLM-designs-chips projects /dog-head). Two uses need distinguishing: the LLM as code generator, and the LLM as search strategy. Measured against the three-part yardstick of 3.5, the two score very differently.

### 5.1 LLMs generating optimized code directly

One of the early landmark works is Cummins et al. (arXiv 2309.07062), *Large Language Models for Compiler Optimization*. They trained a 7B-parameter transformer that takes unoptimized LLVM assembly as input and outputs the best sequence of compiler options along with the optimized code.

A few data points worth recording. During training the model simultaneously predicts instruction counts before and after optimization as well as the optimized code itself; these two auxiliary tasks improved the main task. On the test set the model reduced instruction count by 3.0% relative to the compiler, beating two baselines that need thousands of compilations. The model's generated code compiled in 91% of cases and matched the compiler's output exactly in 70%.

That result indicates the LLM learned the compiler's internal optimization logic to some degree — it isn't "guessing" the output; it absorbed a fair share of the causal relationships at the IR level.

### 5.2 LLM Compiler

Meta's LLM Compiler, released in June 2024, pushed this route to a more systematic position. Built on Code Llama, it continues pretraining on 546 billion tokens of LLVM IR and assembly, then instruction-tunes on a compiler-emulation dataset. Capabilities include code-size optimization (reaching 77% of autotuning's search potential), disassembly (recovering high-level IR from assembly), and optimization-pass recommendation (given IR, recommend the best pass sequence). The model was released under a permissive commercial license.

LLM Compiler represents a paradigm different from RL or search: treat compiler optimization as a **translation problem** — from unoptimized code to optimized code, from assembly to IR — and use the language model's strong prior to accelerate it. This route's search-space expressiveness is theoretically the widest (natural language plus all code), but the cost is losing the symbolic methods' equivalence guarantees: optimization turns into probabilistic guessing.

### 5.3 LLM plus RL: Compiler-R1

A newer direction connects LLMs with RL. Compiler-R1 (arXiv 2506.15701, 2025) trains an LLM agent to select LLVM passes in the CompilerGym environment, using a two-stage end-to-end RL pipeline: SFT on a high-quality reasoning dataset first, then RL with outcome-based rewards. Results show an average 8.46% reduction in IR instruction count over `opt -Oz` across 7 datasets.

Compiler-R1's architecture has two key points. First, it doesn't have the LLM predict the whole pass sequence in one shot; the agent interacts with the compilation environment over **multiple turns** — inspecting the current IR, calling tools (like instrcount), deciding the next action. That extends 5.1's "one-shot generation" into a closed loop with feedback, where the LLM sees the effect of each step before deciding the next, greatly reducing dependence on single-step prediction accuracy. Second, RL training teaches the agent when to use tools and when to stop — something pure SFT can't learn.

That's the early paradigm of agentic compiler tuning: treat the LLM as a tool-using agent that forms a closed loop with the compiler and profiler. In a domain like compilation with "objective ground truth" (runtime, instruction count), the agent loop's usability is far better than in many general agent settings — the reward signal is unambiguous, failures are verifiable, and those two points are enough to make RL training work.

### 5.4 The LLM route's correctness problem

The LLM's biggest bottleneck in compilation is correctness. A language model may output syntactically correct but semantically wrong optimized code — changing floating-point behavior, violating aliasing rules, dropping a volatile marker. Even at 91% correct, a 9% failure rate is unacceptable in a production compiler.

Current mitigation directions:

- **Verifier-guided generation**: filter LLM output with formal equivalence checkers (Alive2, translation validation).
- **The LLM as search strategy rather than generator**: have the LLM output a description of the transformation (e.g., "tile outside the third loop, size 32") and let the trusted compiler apply it. That restricts the LLM's output to a discrete space spanned by compiler primitives, sidestepping the correctness problem of arbitrary code generation.
- **Hybrid LLM and symbolic methods**: the LLM proposes candidate rewrites, the symbolic system (e-graph) guarantees equivalence.
- **RL's outcome reward**: let training-time failure penalties substitute for explicit verification — Compiler-R1 is an example.

All of these are early, but the consensus has converged: **LLMs must combine with some correctness-guarantee mechanism before entering the mainstream compiler pipeline.** The most romantic road — having the LLM generate the final IR directly in one shot — is unlikely to become mainstream in the foreseeable future.

## 6. Summary

A summary table:

![Summary of the four method classes](/assets/images/posts/symbolic-heuristic-and-ai-search-in-ai-compilers/c01afb7487ff05a20da58cbb9cda0fa6.jpg)

From this a few empirical correspondences can be distilled.

**Graph-level rewriting** (DNN compute-graph rewriting) suits symbolic methods. The rule count is limited and compositionality is strong; the e-graph's advantage is most pronounced at this layer. When the e-graph blows up, bring in MCTS or RL guidance.

**Tensor-program / schedule level** remains dominated by heuristics plus a learned cost model. The Ansor / MetaSchedule line is stably usable in industry. RL's extra gain at this layer hasn't yet been shown to justify its complexity.

**LLVM-level pass ordering** suits RL and LLMs. The CompilerGym environment is mature, the problem is thoroughly discretized, and sample efficiency isn't bad. Work like Compiler-R1 shows LLM–RL hybrids progressing at this layer too.

**Algorithm discovery** (the algorithmic level) is currently carried mainly by RL + MCTS (AlphaTensor). It's the deepest layer search can reach and the most expensive, and so far breakthroughs exist only on problems with a clean game structure, like matrix multiplication.

**Cross-layer optimization and new-hardware bring-up** is where the LLM-assisted route may bring change in the next two to three years — the LLM is currently the only search-and-generation mechanism that can digest unstructured documents (hardware manuals, whitepapers). When a new accelerator launches with a few thousand pages of PDF and an early SDK, heuristics and RL have to wait for datasets and cost models before they can start; an LLM can at least read the docs and generate a first runnable kernel.
