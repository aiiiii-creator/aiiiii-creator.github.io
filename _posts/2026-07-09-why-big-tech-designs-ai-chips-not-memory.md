---
layout: post
title: "Why Big Tech Designs Its Own AI Chips but Not Its Own Memory"
date: 2026-07-09 12:00:00
categories: [semiconductors, memory]
excerpt: "This round of AI profits didn't go only to NVIDIA. A few memory makers posted rarely-seen margins too, which raises a question: the cloud giants buy more memory than anyone, so why don't they design their own memory the way they design their own accelerators?"
---

<style>
.tf-fig{ margin:2.4rem 0; }
.tf-card{ border:1px solid rgba(29,27,24,0.12); background:#fffdf9; border-radius:18px; padding:20px 20px 14px; box-shadow:0 20px 60px rgba(38,32,24,0.08); }
.tf-cap{ font-family:"Noto Sans SC",sans-serif; font-size:.86rem; font-style:italic; color:#625a50; text-align:center; margin-top:12px; line-height:1.5; }
.tf-svg{ width:100%; height:auto; display:block; }
.tf-svg .axis{ stroke:rgba(29,27,24,0.35); stroke-width:1.5; }
.tf-svg .bar{ fill:#0f766e; }
.tf-svg .barghost{ fill:#0f766e; opacity:.16; }
.tf-svg .tick{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; text-anchor:middle; }
.tf-svg .ptitle{ fill:#1d1b18; font-family:"Space Grotesk","Noto Sans SC",sans-serif; font-size:14px; font-weight:700; }
.tf-svg .psub{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; }
.tf-svg .lblm{ fill:#0b5f59; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:600; text-anchor:middle; }
.tf-svg .atitle{ fill:#625a50; font-family:"Noto Sans SC",sans-serif; font-size:12px; font-weight:600; }
.tf-svg .divider{ stroke:rgba(29,27,24,0.12); stroke-width:1; }
.tf-stats{ display:flex; gap:14px; margin:2.2rem 0 .6rem; flex-wrap:wrap; }
.tf-stat{ flex:1 1 150px; border:1px solid rgba(29,27,24,0.12); background:#fffdf9; border-radius:16px; padding:18px 20px; box-shadow:0 20px 60px rgba(38,32,24,0.06); }
.tf-statnum{ font-family:"Space Grotesk","Noto Sans SC",sans-serif; font-size:2rem; font-weight:700; color:#0f766e; letter-spacing:-0.02em; }
.tf-statlab{ font-family:"Noto Sans SC",sans-serif; font-size:.9rem; color:#625a50; margin-top:4px; }
.tf-statcap{ font-family:"Noto Sans SC",sans-serif; font-size:.82rem; font-style:italic; color:#8a8175; margin:0 0 2rem; }
.post-content table{ display:block; overflow-x:auto; border-collapse:collapse; font-family:"Noto Sans SC",sans-serif; font-size:.9rem; }
.post-content th,.post-content td{ border-bottom:1px solid rgba(29,27,24,0.12); padding:9px 12px; text-align:left; vertical-align:top; }
.post-content thead th{ border-bottom:2px solid rgba(29,27,24,0.25); font-weight:700; }
.post-content blockquote{ border-left:3px solid #0f766e; }
</style>

This round of AI profits did not go only to NVIDIA. Reading company earnings over the weekend, one pattern stood out: several memory-chip makers took a large share too, with gross margins rising to levels rarely seen in the past. The buyers of most of that memory are the same cloud companies that already design their own AI chips. So the question follows. If memory is this profitable, why don't they design their own memory chips the way they design their own accelerators?

## Memory Is a Cyclical Business; NVIDIA Is Not

Answering the question starts with one structural difference between memory and NVIDIA: memory is a cyclical industry and NVIDIA is not. This difference recurs throughout, so it comes first.

Memory chips have been the most cyclical product in semiconductors for decades, and the pattern is simple. When demand runs hot, supply falls short, prices rise, makers profit, and everyone expands capacity. Once capacity overshoots, prices fall and the whole industry loses money together. The cycle repeats every few years.

> Over more than a decade, the memory industry created almost no net economic value, because the money made in one upcycle tends to fall back out in the next.

The cyclicality here does not mean demand fluctuates; it means margins collapse to zero or below at the bottom of the cycle. In the last downturn, SK Hynix and Micron each lost several billion dollars.

NVIDIA is different. It is close to a monopoly in AI accelerators, and CUDA locks customers in, so its gross margin holds at 70% to 90% year after year rather than earning for two years and giving it back. The market sees this clearly and values the two very differently. NVIDIA's forward PE is around 25×, while memory makers, even during this windfall, trade at 10 to 14×. The gap says one thing: the market believes NVIDIA's profit will persist and memory's profit will be eaten by the cycle.

| | NVIDIA (AI accelerators) | Memory makers (DRAM / flash) |
|---|---|---|
| Market structure | Near-monopoly, CUDA lock-in | Three-way oligopoly, standardized product |
| Gross margin | 70% to 90%, sustained | Collapses to zero or negative at the cycle bottom |
| Forward PE | ~25× | 10 to 14× (even during this windfall) |
| What the market expects | Profit persists | Profit eaten by the cycle |

## Big Tech Already Designs Its Own AI Chips

Start with what has already happened. Google's TPU, Amazon's Trainium, Microsoft's Maia, and Meta's MTIA are all in-house AI chips, built to reduce dependence on NVIDIA and bring costs down.

The same companies buy the most memory. They spend hundreds of billions of dollars a year on AI hardware, and memory is a large part of that bill. If designing their own AI chips saves money, why not apply the same approach to memory? The reason is that designing chips and manufacturing memory are two very different businesses. The four reasons below run from the strongest constraint to the weakest: the first is close to unsolvable, and the last is already loosening.

## First: There Are Chip Foundries, but No Memory Foundries

This is the hardest constraint. Big tech can design its own AI chips because it only designs and does not manufacture. It assembles a design team, draws the architecture, and hands production to a foundry like TSMC. This design-only, no-fab model is called **fabless**. Under it, a company pays only design costs and a one-time tape-out fee to get a chip customized for itself, and the outlay is bounded.

Memory has no such foundry. DRAM and flash use the **IDM** model, where design and manufacturing are bound together and cannot be separated. The value of memory lies mostly in manufacturing process and yield, not design. No neutral factory exists that will fabricate memory for others the way TSMC fabricates logic. So even if a big-tech company wanted to design memory and outsource production, that path does not exist.

There is a second layer here, and it bears directly on whether the numbers work. Because there is no foundry, whoever makes memory carries the entire manufacturing investment. A large share of what memory makers earn has to go back into fabs and EUV lithography machines just to hold position. SK Hynix's plans through 2030 call for tens of trillions of Korean won on Korean fabs and equipment alone, a continuous outlay that never fills up. In-house AI chips are the opposite: under fabless, the only costs are design and tape-out, and they are one-time. One is continuous spending to build fabs, the other is a one-time design fee, and the two are different in kind. This is a direct reason big tech would rather buy memory than build a memory fab.

<figure class="tf-fig">
  <div class="tf-card">
    <svg class="tf-svg" viewBox="0 0 660 300" role="img" aria-label="Fabless one-time design cost versus IDM recurring manufacturing capex over time">
      <line class="divider" x1="330" y1="40" x2="330" y2="272" />
      <text class="ptitle" x="24" y="24">Fabless AI chip</text>
      <text class="psub" x="24" y="42">design + one tape-out</text>
      <text class="ptitle" x="354" y="24">Memory maker (IDM)</text>
      <text class="psub" x="354" y="42">fab + EUV capex</text>
      <text class="atitle" x="18" y="150" text-anchor="middle" transform="rotate(-90 18 150)">capital outflow</text>
      <line class="axis" x1="44" y1="220" x2="308" y2="220" />
      <rect class="bar" x="60" y="92" width="34" height="128" rx="2" />
      <rect class="barghost" x="112" y="210" width="34" height="10" rx="2" />
      <rect class="barghost" x="164" y="212" width="34" height="8" rx="2" />
      <rect class="barghost" x="216" y="213" width="34" height="7" rx="2" />
      <rect class="barghost" x="268" y="214" width="34" height="6" rx="2" />
      <text class="tick" x="77" y="238">yr 0</text>
      <text class="tick" x="129" y="238">1</text>
      <text class="tick" x="181" y="238">2</text>
      <text class="tick" x="233" y="238">3</text>
      <text class="tick" x="285" y="238">4</text>
      <text class="lblm" x="176" y="262">one-time, bounded</text>
      <line class="axis" x1="360" y1="220" x2="624" y2="220" />
      <rect class="bar" x="372" y="102" width="30" height="118" rx="2" />
      <rect class="bar" x="420" y="110" width="30" height="110" rx="2" />
      <rect class="bar" x="468" y="96" width="30" height="124" rx="2" />
      <rect class="bar" x="516" y="106" width="30" height="114" rx="2" />
      <rect class="bar" x="564" y="99" width="30" height="121" rx="2" />
      <text class="tick" x="387" y="238">0</text>
      <text class="tick" x="435" y="238">2</text>
      <text class="tick" x="483" y="238">4</text>
      <text class="tick" x="531" y="238">6</text>
      <text class="tick" x="579" y="238">8</text>
      <text class="lblm" x="492" y="262">recurring, every cycle</text>
    </svg>
  </div>
  <figcaption class="tf-cap">Figure 1. Two different kinds of spending. A fabless AI chip is a bounded, one-time design and tape-out fee; a memory fab is continuous capital that has to be reinvested to hold position, cycle after cycle.</figcaption>
</figure>

## Second: The Manufacturing Barrier Is High

Could a company build its own fab instead? Basically no. An advanced memory fab costs $15 billion to $20 billion and takes two to three years from groundbreaking to production. Harder than the money is the process know-how. The key to memory manufacturing is driving yield up, and much of that experience cannot be written into patents; it takes long practice to acquire.

History makes this clear. The memory industry once had more than a dozen makers. Multiple cycles wiped most of them out, some to bankruptcy, some acquired, until only three remained worldwide: Samsung, SK Hynix, and Micron. A new entrant trying to catch decades of accumulation from zero faces very long odds. Even NVIDIA, which has the money and badly needs memory, does not make its own; it buys directly from these three.

## Third: The Motivation Is Different

Even where it is technically possible, the motivation is weaker, and this is where the cyclicality from earlier comes in.

On the NVIDIA side, it is close to a monopoly with 70% to 90% gross margins, an unavoidable and expensive supplier. In-house AI chips save a lot, and because that supplier's profit is persistent, bypassing it today keeps saving next year and the year after. The savings are predictable, so the investment pencils out.

Memory is a three-way oligopoly with standardized products and prices that swing with the cycle. As the largest buyer, big tech already has bargaining power: it can play suppliers against each other and buy from several at once to spread risk. More to the point, spending tens of billions to build a memory fab means recovering the cost slowly over ten years, but memory is cyclical, so the fab could come online just as the industry turns down and the whole sector loses money. The PE gap from earlier is exactly this. The market will not bet that memory's profit persists, and big tech will not shoulder a fab that might lose money alongside the cycle just to replace a supplier whose profit was going to evaporate cyclically anyway. Between buying and building, the return on investment is far apart.

## Fourth: Memory Is a Commodity, but This Is Loosening

Another benefit of in-house AI chips is customization. A company can co-optimize hardware and software for its own workloads, such as recommendation systems and large-model training and inference, and gain an efficiency edge others do not have.

Memory has long been a commodity. The industry follows a single **JEDEC** standard, memory from different makers is largely interchangeable, the room for customization is small, and the value of co-optimization is low. Buying the standard part is enough, and designing your own is unnecessary.

This is the weakest of the four reasons, and **HBM** is changing it. HBM is the part of memory that is de-standardizing. Marvell is an example. In late 2024 it announced a custom HBM compute architecture, defined and developed together with Micron, Samsung, and SK Hynix. Instead of the standard JEDEC interface, it redesigns the interface between the XPU compute die and the HBM base die.

<div class="tf-stats">
  <div class="tf-stat"><div class="tf-statnum">−70%</div><div class="tf-statlab">interface power</div></div>
  <div class="tf-stat"><div class="tf-statnum">+25%</div><div class="tf-statlab">silicon area freed</div></div>
  <div class="tf-stat"><div class="tf-statnum">+33%</div><div class="tf-statlab">memory capacity</div></div>
</div>
<p class="tf-statcap">Marvell custom HBM against the standard JEDEC interface, up to these figures.</p>

The premise that memory is interchangeable and cannot be customized no longer holds on the most advanced HBM.

## Big Tech Is Already Involved in Memory

Saying big tech does not touch memory at all is inaccurate. The accurate statement is that big tech is getting into the design and definition of memory using the same approach as its AI chips, while staying out of manufacturing.

Marvell's custom HBM above is one case, pulling all three memory makers into a custom architecture driven by cloud-customer demand. The CXL memory standard led jointly by Microsoft, Google, and Meta is another, defining how memory connects and is shared. The involvement goes beyond setting standards. Big tech is locking in capacity and price directly through multi-year contracts. OpenAI has signed a letter of intent with SK Hynix. Micron's full-year 2026 HBM and Kioxia's full-year 2026 NAND are already booked out under long-term contracts, and some customers are signing agreements for 2027 and 2028.

A technical shift shows that even the first reason, no memory foundry, is starting to loosen. HBM's base die was previously made on memory process nodes, which limited performance, and SK Hynix and Micron lack an advanced logic process of their own. So Micron announced that HBM4E will offer some customers a custom base die on TSMC's advanced logic process, and Samsung's and SK Hynix's HBM4 are also working with TSMC. The base-die layer of HBM is being carved out of the memory makers' own territory and handed to TSMC. The rule that memory has no foundry has been torn open on the most advanced HBM.

So big tech does not build its own DRAM fabs not because no one thought of it, but because the barrier is too high and building is not worth it. Its approach is to keep control of design and definition and leave manufacturing to Samsung, SK Hynix, and Micron. This matches how it handles AI chips: do the design and integration in-house, leave manufacturing to others.

## The Interesting Part

Put together, one thing stands out. Memory used to be cyclical in large part because the product was standard and interchangeable, every maker's part the same, leaving customers to compete only on price. What big tech is doing now weakens that interchangeability. HBM is starting to be customized per customer, and big tech is locking volume and price with long-term contracts, so memory is slowly turning from a commodity into a custom product bound to specific customers.

For decades big tech wanted memory to be a commodity, so that suppliers would undercut each other. This time it is reversed: big tech is actively helping memory become non-standard. In the AI era, stable supply and matched performance matter more than a few points off the unit price. The old game used the cycle to push prices down; the new one tries to lock supply in.

Whether memory can truly escape the cycle through HBM and long-term contracts, this record-setting earnings report cannot yet show. That will be clear only when the industry next turns cold.

> Big tech not designing its own memory was never an oversight. Making memory and designing chips are simply two different businesses.
