---
layout: post
title: "Lip-Bu Tan's Two-Tape-Out Rule Meets the LLM"
date: 2026-05-25 12:00:00
categories: [semiconductors, eda]
excerpt: "The rule only works because a failed tape-out can now be traced to individual lines of RTL and individual engineers. LLM-generated code is quietly breaking that chain — and the fix is a second generation of formal verification."
---

*The rule only works because a failed tape-out can now be traced to individual lines of RTL and individual engineers. LLM-generated code is quietly breaking that chain — and the fix is a second generation of formal verification.*

Lip-Bu Tan's new rule — "A0 must go straight to production, B0 can still save you, after that you're fired" — sounds harsh, but the precondition for being that harsh is that the chip-design flow has reached a threshold: responsibility for a failed tape-out can be traced to individual lines of RTL and individual engineers. That is a direct result of fifteen years of EDA-tool evolution, and the technical basis on which the rule can stand institutionally.

*This is a translation of a Zhihu answer originally published on May 25, 2026.*

But there's an irony buried here. Precisely when accountability can be made that precise, LLMs are entering the chip-design flow at scale, and that precise chain of accountability is quietly coming apart.

### Pentium FDIV

Around Christmas 1994, a bug was found in the floating-point divider of Intel's Pentium. The cause: the lookup table used by the SRT division algorithm was missing 5 entries, so for certain inputs the division result was wrong in the 4th to 5th significant digit. The origin of the bug is classic: the table was filled in by hand, the verification team's test vectors didn't cover those 5 grid points, and all routine verification passed.

The affair cost Intel $475 million, sank the stock, and had the CEO apologize publicly. The deeper effect was that it directly spurred the widespread adoption of formal verification in chip design. From the late 1990s, formal verification of high-risk modules — floating-point units, cache-coherence protocols, critical datapaths — became industry standard. Intel itself used Pentium FDIV as an internal cautionary tale for over twenty years.

The lesson has one line: when a bug sits entirely in the blind spot of routine verification, no review process, however strict, catches it; what's needed is a verification method that doesn't depend on test coverage but proves correctness mathematically.

### Precision

A chip going from RTL to GDS to tape-out passes dozens of sign-off checkpoints: post-synthesis equivalence, formal verification, timing sign-off, power sign-off, DFM, DFT, ESD, metal density, antenna effect, and more. Each generates a traceable report.

If A0 has a problem, localization runs roughly like this: post-silicon debug catches the faulting signal through scan chains and on-chip trace, maps it back to RTL, checks git blame for who wrote it, when, and which reviews it passed, then checks the verification team's coverage report for why it wasn't caught, and finally the sign-off waiver list to see whether some engineer signed it through. A bug can usually be traced to one or two specific engineers plus the few signers on the review chain. That accountability mechanism has matured to where he can trust the outcome is fair with his eyes closed. This isn't the old era of "tape-out failed, everyone shares the blame."

But LLMs tear that chain open. LLM-assisted RTL generation, testbench generation, and bug fixing are already rolled out at scale inside NVIDIA, Cadence, and Synopsys. The problem is that LLM-produced code has a trait the traditional verification flow handles poorly: it looks right, passes every routine check, and fails on certain corner cases — and its failure modes differ from a human's.

Allocating responsibility gets thorny too. When an LLM generates a piece of problematic RTL under some prompt, the reviewing engineer doesn't spot it, it passes lint and synthesis, and the tape-out fails — who's responsible? The engineer who wrote the prompt? The reviewer? The LLM tool vendor? The open-source repos in the training data? There's no industry consensus. I think this is the first structural challenge Tan's hard accountability regime meets in the LLM era.

### The LLM era needs second-generation formal verification

Back to the rule. If A0 must go straight to production, the hidden bugs LLM-assisted design introduces are the rule's biggest execution risk. Over the next three to five years, I'd guess several things appear.

First, mandatory formal verification for LLM-generated code. Any LLM-generated RTL has to pass equivalence checking against a human-written or formally specified reference model before entering review; if it fails, it doesn't proceed.

Second, provenance tracking. Every line of RTL is tagged human-written or LLM-generated, and the LLM-generated portion is held to a stricter verification standard. Somewhat like license tracking in open source today, but at RTL line granularity.

Third, dedicated checkers for LLM failure modes. LLM errors have recurring patterns — boundary-condition handling, incomplete state-machine coverage, violated implicit assumptions — that can be captured by tooling.

Fourth, possibly "dual-source verification." Critical modules written once by an LLM and once by a human, then equivalence-compared, with differences arbitrated by a third-party reference. Somewhat like N-version programming in aviation and nuclear, lifting redundancy from the hardware layer to the design-source layer.

### Tan is betting on a tool upgrade

His iron-fisted policy made complete sense in the pre-LLM era: clear accountability chain, mature tools, a single responsible party. But he's hit the inflection where LLMs enter the design flow at scale, and for the policy to keep working, the supporting verification system has to upgrade.

Otherwise a new awkwardness appears: an A0 fails, accountability traces to a piece of LLM-generated code, and the engineer can't prove whether they should or shouldn't have caught it in review — there's no mature industrial standard for that boundary. In that situation the legitimacy of the "fire" decision itself gets challenged.

Tan's policy is essentially using management to force tool evolution. Once chip design goes from "human plus EDA" to "human plus EDA plus LLM," the traditional verification system falling behind is inevitable. Whoever industrializes LLM-era formal verification first can genuinely support a target as aggressive as A0-to-production. Pushing Intel to the front line of this new rule is itself Tan betting that the tool upgrade happens within his tenure.

For chip engineers, the subtext is clear too. People who can write RTL will compete with LLMs for their jobs; people who can use LLMs to write RTL will beat the first group on efficiency; but the ones who really won't be fired are those who can review LLM output, design verification strategies aimed at LLMs, and clearly locate the boundary when accountability comes calling. Tan's policy won't show mercy, but what he really wants to keep is that third kind of engineer.
