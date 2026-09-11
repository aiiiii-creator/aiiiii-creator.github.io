---
layout: post
title: "Can the UK Be No. 2 in AI Chips? Notes from a Nick McKeown Talk"
date: 2026-05-05
categories: [semiconductors, industry]
excerpt: "I assumed the title was British irony. It's an actual national strategy: skip the fabs, own design, inference, and photonics, and ship 50 UK-designed AI chips in five years. What's real about it, and what isn't."
---

*I assumed the title was British irony. It's an actual national strategy: skip the fabs, own design, inference, and photonics, and ship 50 UK-designed AI chips in five years. What's real about it, and what isn't.*

Today I heard Stanford professor Nick McKeown give a talk at our university, "How the UK can be No. 2 in the world in AI chips." I assumed it was that peculiar British self-deprecating irony; it turned out to be a seriously considered strategy. The topic is interesting: everyone talks about China's domestic substitution and getting out from under the US chokehold, but other countries have similar ambitions — it's just that their target may be China.

*This is a translation of a piece originally published on Zhihu on May 5, 2026.*

My personal feeling, from hunting for internships lately, is that AI-chip architecture jobs in the UK are very scarce — maybe only Arm, Graphcore, and Imagination are relevant (and the latter two often have no openings…). So I wondered what he'd use to challenge mainland China plus Taiwan for the position of the world's No. 2 AI-chip nation. But after the talk I thought there was a fair amount of substance, and it makes a good entry point for discussion. Whether "how can be" means "how to get there" or "why it's already there" is a matter of taste \dog-head.

Nick is now based in the UK, sits on the Prime Minister's Council for Science and Technology (CST) and is a non-executive director of ARIA, and this July he was lead author of a report to the government, *Building a Sovereign AI Chip Design Industry in the UK*. The "be No. 2 in the world" framing, I think, is more a traffic-generating pitch to politicians; precisely, it's a carefully calculated differentiated positioning — don't compete with the US and China in the capital-heavy fab game, but focus on chip design, AI inference, and photonics: 50 UK-designed AI chips within 5 years, and closing a gap of 12,000 chip designers. He recommended to the PM six departmental measures (talent, photonics, civil–military coordination, cross-pipeline investment, opening infrastructure, international process access) plus ARIA Scaling Compute funding to drive it, turning "be No. 2" from self-mockery into formal national strategy.

On China, he said we're confined to the 7 nm range with yields still not great. In fact China's and the UK's AI-chip industries show completely different development paths. China takes the heavy-asset route of "national mobilization + giant market + large-scale domestic substitution," forced by US export controls; Huawei Ascend, Cambricon, Hygon, Moore Threads, MetaX, Biren, Enflame, T-Head, Kunlunxin, Horizon, Black Sesame, and a dozen-odd others ramped rapidly in 2024–2026, with domestic share expected to rise from 17% in 2023 to 55% in 2027; Huawei's Ascend 910C shipped 700,000–1,000,000 units in 2025 and is the most successful AI-chip product so far.

Taiwan needs no elaboration; it plays a core role in the AI-chip supply chain (TSMC, ASE, and MediaTek in particular).

The UK takes the light-asset route of "design/IP + advanced research + sovereign compute," lacking advanced wafer manufacturing but holding Arm (he also mentioned the rapidly rising CPU-to-GPU ratio — I wonder if he's a stock investor too), Imagination (GPU IP), Graphcore (IPU, acquired by SoftBank in July 2024 for about $500 million), XMOS (edge AI), Fractile (in-memory compute, rumored to have OpenAI investment recently), and others.

As for UK universities, the few I'd rate as so-so — awkward against North America; lately the paper count at the top four architecture venues doesn't seem to match Tsinghua, PKU, Fudan, and SJTU, let alone the expensive ISSCC, and startup incubation doesn't match the CAS Institute of Computing Technology; overall a position of good but not top-tier strength. But research here may be a little less careerist — in a direction like compilers, for instance, quite a few people settle down and work seriously (a plug for our ICSA group at Edinburgh, haha).

Policy and funding go without saying; I can't be bothered to cite numbers. In short: the UK is making the design slice of the cake well, and taking whatever slice it can…
