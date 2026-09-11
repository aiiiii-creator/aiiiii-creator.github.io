---
layout: post
title: "Cerebras Before the IPO: A Fundamentals Read"
date: 2026-05-10
categories: [semiconductors, cerebras]
excerpt: "52× sales, 86% of revenue from two UAE customers, a GAAP profit that is an accounting artifact, and a latency edge with a limited window. A look at CBRS a week before it lists."
---

*52× sales, 86% of revenue from two UAE customers, a GAAP profit that is an accounting artifact, and a latency edge with a limited window. A look at CBRS a week before it lists.*

**Important disclaimer:** this piece is compiled from public information and analogies to comparable historical events, for research and discussion only. It is not investment advice.

*This is a translation of a piece originally published on Zhihu on May 10, 2026.*

Cerebras lists on Nasdaq next Thursday (May 14) under the ticker CBRS: 28 million shares at $125–$135, a valuation of up to $28.5 billion, oversubscribed 20×. It may be the largest US tech IPO so far this year. I analyzed the architecture earlier, in [LLM Decode Accelerators, Part 1: The Cerebras Wafer-Scale Engine](https://zhuanlan.zhihu.com/p/2026781196298757013).

That valuation is 1.2× its funding valuation of three months ago, and 52× its 2025 revenue — twice as expensive as NVIDIA. Time to look at the fundamentals as a whole.

## 1. Product competitiveness

This is where the whole valuation story starts. The differentiation is ultra-low inference latency.

On the same model, an NVIDIA B200 cluster generates 1,000 tokens per second per user; Cerebras does 2,500. Response time gets pushed under 50 ms, where GPU clusters typically take 200–300 ms. SemiAnalysis's third-party testing found the CS-3 up to 21× faster than a DGX B200 and 32% cheaper on certain inference tasks. Plenty of scenarios today love low latency: agents, voice assistants, real-time code completion. 50 ms and 200 ms are fundamentally two different things — the former can carry a fluent voice assistant, the latter can't. That's the main reason for a 52× P/S.

But Cerebras isn't alone on the low-latency inference track. Groq takes the LPU + pure SRAM + deterministic scheduling route, with product maturity close to Cerebras's, and was absorbed by NVIDIA for about $20 billion in early 2026. SambaNova was the pioneer in the same tier — valued at $5 billion back in 2021 — but commercialization never took off; it went through large layoffs and a pivot in 2024–2025 and has basically exited mainstream competition. Its existence is itself a warning to Cerebras: a technical lead doesn't solve customer concentration.

Beyond those, there's the hyperscalers' general-purpose compute route and the TPU route. General-purpose compute is "training plus inference, one-stop procurement." AMD's MI450 is a scale-crushing competitor: OpenAI 6 GW, Meta 6 GW, a 50,000-chip Oracle cluster — well over a hundred billion dollars of contracts signed in total. NVIDIA's Vera Rubin has also proactively raised its on-chip SRAM share to answer the inference-latency question. AWS has its own Trainium 2 and is building Trainium 3; Google has TPU v8.

And there are newer players on even more aggressive routes. Etched hard-wires the Transformer architecture into silicon (it can only run Transformers); it raised $500 million at a $5 billion valuation in January 2026, but Sohu hasn't shipped and every number is self-reported. Taalas goes further than Etched, printing the model weights themselves into the chip's metal layers (each chip runs exactly one model); its CEO is Tenstorrent founder Bajic (with an AMD stint too — it really is all one circle), and its first chip, HC1, measured 10× faster than Cerebras, but it currently tops out at 8B-parameter models and has no major customers. Both set a ceiling on the differentiation premium of the "low-latency inference" track. If either reaches scale, Cerebras's 50 ms lead gets diluted.

A summary table:

![Competitive landscape, summarized](/assets/images/posts/cerebras-fundamentals-1-before-the-ipo/b1fa88cf8c14c6c1c2496fd2e51fe91a.jpg)

**So I think the differentiation window is limited.** Etched's HC2 delivering half of what it promises, Taalas's HC2 ramping, Vera Rubin shipping in volume, Trainium 3 catching up — all quite possible. Any one of them compresses the advantage.

## 2. Order book and financials

Total backlog is $24.6 billion, in three main pieces.

The largest is OpenAI's Master Relationship Agreement: a little over $20 billion in total, a three-year term, locking in 750 MW of inference compute, with terms allowing expansion to 2 GW. It was signed at the end of 2025, and at the same time OpenAI lent Cerebras $1 billion, secured by warrants for 33.4 million shares. OpenAI is thus simultaneously the largest customer, a creditor, and a potential shareholder.

The second piece is two long-standing UAE customers. MBZUAI was 62% of 2025 revenue and G42 24% — 86% combined. The prospectus doesn't separately disclose their remaining backlog; back-solving from the 2024–2025 recognition pace puts it at roughly $3–4 billion. The sheikhs have money, and if the local deployments work well they'll presumably keep buying.

The third is a binding term sheet with AWS Bedrock. Note: a term sheet, not a definitive contract. If it lands, calling Cerebras compute becomes one of the default options for Bedrock users — Cerebras's first genuine entry into a US hyperscaler's inference-serving stack. But there is no public signing timeline, and this piece isn't counted in RPO.

2025 gross profit was $199 million on $510 million of revenue — a 39% gross margin. The historical curve: 12% in 2022, 33.5% in 2023, 41.1% in H1 2024, back down to 39% in 2025. **The gross margin is actually low, and that's the economics of the wafer-scale route itself: you sell a whole wafer at a time, so revenue per unit isn't as high as other chip companies', and once yield is factored in, overall margins are well below other design houses'.**

The recognition schedule is in the S-1: 15% recognized across 2026–2027 combined, about $3.7 billion; 43% in 2028–2029, about $10.6 billion; the remaining 42% after 2030. Converting that into annual revenue: 2025 $510 million; 2026 in the $0.9–1.2 billion range; 2027 $1.8–2.2 billion; 2028 jumping to $3–4 billion — the last two contingent on OpenAI's 750 MW coming online as planned in the second half of 2027. Which means the 2026 financials won't carry a "revenue tripled" narrative. 15% of RPO spread over two years, plus the UAE orders winding down, gives full-year revenue of a bit over $1 billion. The real revenue inflection may come in 2028.

So the GAAP net income of +$87.9 million — flipping from −$485 million in 2024 within one year — sounds as though Cerebras is already profitable, but it's an accounting phenomenon, not an operational improvement. The same prospectus shows a Non-GAAP operating loss of $146 million and a Non-GAAP net loss of $75.7 million. Most of the $574 million swing comes from three non-cash accounting items: reversal of liabilities from an early forward-share contract with G42, fair-value changes on the OpenAI and Amazon warrants, and stock-compensation adjustments. Strip those out and the real picture is a high-growth company with $510 million of 2025 revenue, growing 76%, still burning $150 million of cash a year — and with US domestic revenue actually down 34%, all of the growth carried by two Middle Eastern customers.

**So I think Cerebras has to diversify its customer base within 18 months; otherwise, technical lead or not, the valuation could go the way of SambaNova's.**

## 3. Ownership structure

Cerebras's shareholder structure has several unusual features.

The most striking is AMD. AMD only came in at the February 2026 Series H, a single round, with the amount undisclosed. But this isn't an ordinary financial investment: founder Andrew Feldman and chief architect Michael James founded SeaMicro in 2007, sold it to AMD for $334 million in 2012, and Feldman spent three years inside AMD before leaving to found Cerebras. That thread was never cut. AMD is simultaneously a Cerebras shareholder and a direct competitor. Read it either way, but at minimum it says AMD itself believes the wafer-scale route can't be displaced by GPUs in certain inference scenarios — or it's a speculative investment, learning from big brother NVIDIA.

The speed of the valuation jumps is worth noting too. Series G in September 2025: $1.1 billion raised at $8.1 billion post-money. Series H five months later: $1 billion raised at $23 billion post — a 2.84× jump. What happened in between was the $20 billion OpenAI contract. Tiger Global was in both rounds, following in G and leading H; Fidelity took the same path. That "see the big order, immediately double down" rhythm back-solves to how Tiger and Fidelity themselves price the probability of the OpenAI contract being fulfilled.

The absence of big-tech capital is more notable than who came in. OpenAI's shareholders include Microsoft; Anthropic has Google and Amazon; Groq has BlackRock. On Cerebras's side — NVIDIA, Microsoft, Google, Amazon — not one. The only big-tech investor is AMD. AWS is a customer, holding warrants rather than equity. Read it either way: on one side, Cerebras lacks a big-tech ecosystem as endorsement, and its binding to the hyperscalers is shallower than Anthropic's; on the other, Cerebras is more commercially independent, not hostage to any one giant's strategic choices.

G42's treatment is another point of interest. G42 was previously both a major customer and a shareholder, and the first IPO attempt in 2024 was blocked by CFIUS precisely over the G42 relationship. In this relaunch, G42's equity has been restructured into non-voting shares and the prospectus no longer lists G42 as an "investor," but the actual holding remains. Inside the company this is called sanitization — the economic interest stays, the governance rights are stripped. CFIUS's clearance this time rests on accepting that structure. But be clear: G42's money is still in the company; it just has no vote.

The early VCs' paper gains are an account that has to be settled after the IPO. Foundation, Benchmark, and Eclipse all came in at the 2016 Series A, when Cerebras was valued under $100 million. At an IPO valuation of $26.6 billion, all three are sitting on gains of two to three hundred times or more. The 180-day lockup expires in November 2026, right as Cerebras reports Q3 — and by the recognition pace above, annualized quarterly revenue will most likely still be a bit over $1 billion, nowhere near the inflection.

OpenAI's warrants deserve a separate mention. OpenAI doesn't currently count as an "investor," but it holds warrants for 33.4 million shares, roughly 13% dilution on exercise. That equity isn't in the float today and isn't counted in lockup statistics, but OpenAI can exercise unilaterally at any point after the IPO. That's another hidden source of dilution, on a timeline the company doesn't control.

## 4. Impact on other stocks

Everything below is my own guess and extrapolation; it isn't investment advice and I'm not responsible for share prices — between fundamentals and short-term prices lie the randomness of sentiment and flows. Take it as a way of thinking.

The Cerebras roadshow will hammer on "on-chip SRAM replaces HBM," a marginal negative for the HBM narrative, but tiny in volume; NAND and HDD have no product overlap with Cerebras. Though the memory sector has run so far this year that anything could be a negative… Cerebras will keep comparing "on-chip fabric 214 PB/s vs. NVLink 57.6 GB/s," and the short narrative will use it to paint "wafer-scale reduces demand for optical interconnect." But the fundamental negative is small — Cerebras itself is working with Ranovus on wafer-scale CPO, and multi-rack interconnect still needs fiber. NVIDIA, big brother, doesn't care; it has its inference market poached daily and hasn't moved much lately. AMD is both a holder and a rival — hard to say whether that's good or bad.
