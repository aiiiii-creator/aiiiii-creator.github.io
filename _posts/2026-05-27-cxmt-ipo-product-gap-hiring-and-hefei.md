---
layout: post
title: "CXMT's IPO: The Product Gap, the Hiring Spree, and Why Hefei Is the Real Stock God"
date: 2026-05-27
categories: [semiconductors, memory]
excerpt: "China's largest DRAM maker clears its STAR Market listing. Where its HBM actually stands against SK hynix, why it hires our whole department, and how state capital changes the math in a brutally cyclical industry."
---

*China's largest DRAM maker clears its STAR Market listing. Where its HBM actually stands against SK hynix, why it hires our whole department, and how state capital changes the math in a brutally cyclical industry.*

Split into a technical part and a fundamentals-and-policy part; read whichever interests you.

*This is a translation of a Zhihu answer originally published on May 27, 2026.*

### 1. Products

As usual, the products first:

![Product comparison](/assets/images/posts/cxmt-ipo-product-gap-hiring-and-hefei/1c3b018fccfc2b39be6ee40b664ba118.jpg)

Product competition is first of all technical competition. Frankly, CXMT's most advanced products today can't match Samsung's and SK hynix's, but rather than simply counting how many years behind on process node, it's more meaningful to return to the product metrics that actually matter in the LLM era: bandwidth and capacity.

Laying the two out:

Bandwidth (per stack):

- SK hynix HBM3E 12-Hi (volume production Q3 2024): 1.22 TB/s
- HBM4 12-Hi (2026 volume): 2.0–2.5 TB/s
- CXMT HBM2, small-volume production (H2 2024): ~410 GB/s
- CXMT HBM3: ~819 GB/s

Capacity (per stack):

- HBM3E 12-Hi: 36 GB; 16-Hi: 48 GB
- HBM4: 36–48 GB
- CXMT HBM3 mainstream: 16–24 GB
- CXMT's D1z node: about 16 Gb per die, with die area about 40% larger than SK hynix's 1β node at the same capacity

Translated into generations: bandwidth is one to one-and-a-half generations behind (HBM3 vs. HBM3E/HBM4), capacity one generation behind (24 GB vs. 36/48 GB).

But I think 3D DRAM may be a chance to overtake on the curve. Conventional DRAM makes density in the wafer's X–Y plane (shrinking the cell), a road blocked by EUV. 3D DRAM stacks cells in Z (like 3D NAND going from 64 layers to 232 to 500). That happens to be the process direction where China is relatively least weak. YMTC has reached 232 layers in 3D NAND, among the world's leaders, with industrialized experience in etch and ALD. In other words, CXMT's biggest advantage in 3D DRAM isn't how strong its own DRAM process is; it's that the know-how China has proven on 3D NAND can transfer laterally. Samsung, SK hynix, and Micron all plan 3D DRAM volume around 2030; at that window everyone **theoretically** starts from the same line.

![3D DRAM](/assets/images/posts/cxmt-ipo-product-gap-hiring-and-hefei/cxmt-3d-dram.jpg)

Counterpoint Research's November 2025 report gave the first public estimate of CXMT's HBM3 yield: 10–20% — early engineering-production level for HBM3.

That looks low, but it needs context:

- Samsung's early HBM3E yield was also only 30–40%, and industry estimates for SK hynix's HBM3 at first production in 2022 were only 30–50%.
- HBM3's "basically usable" commercial yield threshold is about 50–60%.
- HBM3E 12-Hi's "economically viable" threshold is about 60–70%.

So CXMT's current yield is 30–50 percentage points from "commercially acceptable," a 1–2-year climb. Referencing CXMT's own DDR5 yield curve:

- DDR4 early production: 20–30% (EET-China, December 2024)
- DDR4 mature: about 90%
- DDR5 early (2024): about 80%
- DDR5 mature target (end of 2025): about 90%

From DDR4 to mature yield took CXMT about 4–5 years. HBM3 is far more complex than DDR5 (TSVs, stacking, hybrid bonding), and the climb may be slower. Optimistically, commercial yield (50%+) by 2027–2028; pessimistically, 2029.

But there's a notable counter-signal: in April 2026, Taiwan's TechNews reported that CXMT had begun mass-producing 12-high HBM, and although "yield remains below Korean competitors," "the focus is on fully meeting domestic Chinese AI product demand." That's a distinctive phenomenon — CXMT charging the market with capacity (60,000 HBM wafers a month) and price before yield is in place. It's the typical Chinese-vendor play of "take the position first, optimize later," completely unlike the Korean and American vendors' "wait for yield, then ramp."

### 2. Campus hiring

During last fall's recruiting I noticed something odd: CXMT was hiring people from our school almost indiscriminately. Whatever your direction — circuit design, process integration, materials — as long as the résumé passed, interviews were basically green lights all the way (the interviews were easy), and the offered salary was clearly above the industry average, especially generous for master's and PhD students. Among classmates in the same cohort, the density of people going to CXMT was visibly high. They could even offer Shanghai salaries in Hefei.

My guess is that DRAM's core blocks — the cell array, sense amplifiers, wordline drivers, TSVs — are things only Samsung, SK hynix, and Micron hold complete design-process-integrated know-how for. That know-how was ground out over a dozen-plus years by generations of engineers through trial and iteration on the line; it can't be replicated quickly by buying patents, buying equipment, or poaching a few executives. The only way is to have lots of sufficiently bright young people run it through from scratch.

And CXMT faces an urgent generational gap. Its mainstream G4 node corresponds roughly to Samsung's and SK hynix's 2021 level, 3–4 years behind. Catching up to HBM3, HBM3E, and then 3D DRAM requires staffing across 7–8 directions at once — process integration, an HBM program (controller + TSV + hybrid bonding), materials, equipment.

Cumulative R&D spend from 2022 to H1 2025 was RMB 18.867 billion; the H1 2025 R&D ratio was 23.71% (far above Samsung's 11.74%, Micron's 10.66%, SK hynix's 7.39%); R&D headcount 6,259 (32.43% of staff). Interestingly, founder Zhu Yiming personally transferred 768 million shares for employee incentives (50% of the 1.536 billion shares the board granted Zhu from the second employee stock plan in May 2024, which Zhu then redirected to employee incentives — essentially a pass-through), with a 10-year full lockup plus a 10-year gradual sell-down — the largest personal equity incentive in A-share history, essentially a response to the talent war with the Korean and American vendors. **But I hope CXMT doesn't, like a certain company, lay off large numbers of people before listing to dress up the financials; as a state-backed company it should take on the responsibility of developing and standing by its employees.**

### 3. Policy

CXMT's STAR Market IPO cleared review on May 27, 2026, seeking to raise RMB 29.5 billion; by Q4 2025 DRAM sales its global share is already 7.67% (this figure is uncertain — some say 5% — but that's the order of magnitude). The strategic significance of the IPO isn't just the RMB 29.5 billion (really not much — about one quarter's earnings); it's probably that CXMT's listing activates a re-rating of the A-share memory, equipment, and materials chains and creates a capital cycle around the "domestic substitution" theme. **After all, if money can't be invested abroad, creating some good targets at home is a fine thing (dog-head).**

If technical catch-up is CXMT's visible problem, the memory cycle is its hidden one. DRAM is the textbook strong-cycle industry. From the 1980s to now, almost every 3–4 years: demand expands → vendors add capacity → overcapacity → prices collapse → vendors cut or go bankrupt → supply tightens → prices soar, and around again. This has sent countless DRAM makers to the grave over 40 years, and it's the memory bears' thesis today. Today's memory companies rely on long-term contracts to escape this bind, but our country's particular circumstances should give it more advantages:

CXMT's shareholders are mainly national strategic capital answerable to a 10-year cycle. CXMT lost a cumulative RMB 31.8 billion in 2022–2024, and the shareholders didn't force a change; they added investment. This ability to "afford the loss, hold the line" is itself a core competency in a long-cycle industry like DRAM. And China's DRAM demand is about a third of the world's, with a domestic share long under 5%; domestic substitution is itself a growth curve independent of the global cycle.

Finally, the signature successes of Hefei state capital: BOE, CXMT, NIO, iFlytek, Visionox, Nexchip. The Hefei government builds a complete ecosystem around one industry (displays → panels → glass substrates → polarizers; memory → DRAM → equipment → materials; EVs → vehicles → batteries → motors). And it lifts the home region's industry and employment. (**Could the Jiangxi city-investment folks next door learn a thing or two? Jiangxi people all go elsewhere to build tech companies; nobody comes back to build one at home. Reflect.**)

### **Hefei's city-investment arm is the real stock god.**
