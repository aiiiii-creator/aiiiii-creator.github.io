---
layout: post
title: "Samsung's zHBM: Near-Memory Compute Is the Route That Actually Ships"
date: 2026-08-05 12:00:00
categories: [semiconductors, memory]
excerpt: "Three ways to fight the memory wall, and why the one that changes nothing in software wins. Bandwidth goes from perimeter to area, pJ/bit drops by an order of magnitude, and memory becomes an ASIC business — if customers buy it."
---

*Three ways to fight the memory wall, and why the one that changes nothing in software wins. Bandwidth goes from perimeter to area, pJ/bit drops by an order of magnitude, and memory becomes an ASIC business — if customers buy it.*

FMS was just held and all kinds of information came out, much of it tied to the hot memory stocks. I think near-memory computing is one of the research directions most likely to ship soon — I have papers in the area — so let me look at the parameters and analyze.

*This is a translation of a Zhihu answer originally published on August 5, 2026.*

![](/assets/images/posts/samsung-zhbm-near-memory-compute-ships/60755d2a0c0b3b39ef20c614a6bc4cce.jpg)

There have only ever been three ideas for beating the memory wall: make the interface faster, move compute into memory, or move memory next to compute. The first runs into physics — raising pin rates needs stronger drivers and equalization, energy per bit barely improves, and how many stacks HBM can hang off a die is hard-limited by the die's perimeter.

The second is in-memory compute, PIM, promised for over a decade. Samsung announced HBM-PIM back in 2021, and five years on it's still an exhibit on a show floor, the key limits being tape-out and volume production (though PIM may ship at the edge) and the software demands: it requires you to rewrite kernels and change the compiler stack, and it only helps a minority of memory-intensive operators. Getting an entire software ecosystem to change architecture for one kind of memory has almost never succeeded (Google's software team is nearly bigger than its hardware team).

The third is different. Near-memory compute's gain comes entirely from physical distance: data goes from traveling a dozen-odd millimeters of interposer wire to a few microns of vertical bonding. It doesn't require changing a line of code; it's completely transparent to the layers above. In other words, **it reduces a software-ecosystem problem to a pure packaging-and-materials problem** — and pure hardware problems are exactly the kind the semiconductor industry has been best at solving for fifty years.

**1. It's the only route that gives an order-of-magnitude improvement in pJ/bit**

Today, data from HBM to GPU goes base die → microbump → a dozen-plus millimeters of interposer wire → the PHY driver at the GPU's edge, at an energy cost of a few pJ/bit. Switch to hybrid-bonded vertical interconnect and the distance is micron-scale, close to the energy of on-die wiring.

The key point: **every other route is flat or worse on this metric.** Raising pin rates needs stronger drivers, equalization, and termination, and every HBM generation's per-bit energy improvement is limited. I think this hits the commercial pain point. Today's AI data centers are power-limited, not silicon-limited. Rack power, power delivery, and cooling are what can't scale. Under that constraint, pJ/bit is the real hard currency, and near-memory compute is the only route that improves it by an order of magnitude. Samsung's deck says the route can reach "3× energy efficiency" at best.

**2. From perimeter to area**

This is the most fundamental argument. Today, how much bandwidth an accelerator can hang depends on how much shoreline the die edge has for PHYs and how large the interposer can be (CoWoS's reticle limit). Perimeter grows linearly with edge length; area grows with its square. Stack directly on top and bandwidth scalability goes from L to L².

That's not a quantitative improvement; it's a change of dimension. Samsung's "8× single-GPU performance" mostly comes from here — not a single stack 8× faster, but a whole tier more total bandwidth that can be attached.

![](/assets/images/posts/samsung-zhbm-near-memory-compute-ships/6d65d6b19b889e5689d436c73b768ccb.jpg)

## Commercially — though this is a slideware product, I think it reflects Samsung's overall thinking

### 1. Memory goes from commodity to custom part, and the valuation logic follows

That shift has already begun: HBM4's base die moved to a logic process (Samsung uses its own 4 nm; SK hynix goes to TSMC), the first time memory makers have to do logic design in their product. zHBM pushes that road to its end — custom IP integrated into the interlayer between memory and accelerator, letting system designers tailor capacity and accelerator functions to specific workloads. Memory is no longer something priced per GB; it's closer to an ASIC business, with customer stickiness, bargaining power, and a completely different margin structure. SK hynix's re-rating over the past two years is essentially trading this logic: from cyclical toward "platform asset."

### 2. Packaging goes from cost item to product-definition layer

SemiVision's read is that Samsung is effectively proposing that the next phase of AI-semiconductor competition will be defined by the joint optimization of memory, logic, bonding, packaging, thermal management, and system architecture — which pushes advanced packaging from a manufacturing bottleneck to the core of product design.

In money terms:

- **Demand for giant interposers is in question.** CoWoS is both the industry's hard bottleneck and the source of TSMC's pricing power. If the zHBM route holds, what's needed is wafer-level bonding rather than large-area interposers. TSMC isn't necessarily the loser (it has SoIC and is pushing 3D stacking), but the shape of the moat changes.
- **Hybrid-bonding equipment is the certain beneficiary.** Whether Samsung's zHBM wins, TSMC's SoIC wins, or HBM5/HBM6 switches to bonding internally, Cu–Cu direct bonding is the common path. It's one of the few bets that doesn't depend on which route wins.
- **KGD (known-good-die) testing becomes far more important.** No rework after bonding means testing has to move forward to the bare-die stage. That link is badly underrated right now.

### 3. For Samsung itself: a change of track, not an upgrade

Worth saying plainly: Samsung is the follower on HBM's current curve, and it's hard to turn the tables along that line. Kim Kyung-ryun's "Samsung is back" in the keynote came with a concept model, not a product — which itself says something.

Samsung's one structural advantage is being the world's only IDM with memory, foundry, and advanced packaging all in house. zHBM is precisely the scheme that maximizes that advantage — it requires memory and logic co-design, which only an IDM can do end to end alone. So this isn't a technology announcement; it's an **attempt to rewrite the rules**: if the competitive standard shifts from "whose memory is faster" to "who can optimize memory and logic together," Samsung's lag is reset to zero.

The risk is the same: if customers don't buy it, zHBM is the second HBM-PIM — announced in 2021, still a show-floor exhibit five years later.

Finally, compare zHBM with the recent HBF (see my piece [HBF: What the Spec Says About Chip Architecture, and About Escaping the Cyclical Bin](https://zhuanlan.zhihu.com/p/2068137669847208978)) and you can see the two companies' different thinking.

HBF explicitly takes the OCP open-standard route, with Google and Tenstorrent already in the consortium, and uses UCIe, an open high-speed interconnect, to attach to GPUs and CPUs. SK hynix's line is "expanding the boundary of memory and storage, building a new architecture."

zHBM goes the opposite way: deep binding, single supplier, no rework. NVIDIA's historical preference is to treat memory as a JEDEC-standard commodity it can price across three vendors; it has no motive to let memory makers into its value chain. The in-house ASIC camp (Google's TPU, Broadcom, AWS) is more likely to accept it, since they're already doing deep customization and multi-source price comparison isn't in their procurement logic.
