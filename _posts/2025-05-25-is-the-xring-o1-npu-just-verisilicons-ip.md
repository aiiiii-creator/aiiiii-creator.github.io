---
layout: post
title: "Is the Xring O1's NPU Just VeriSilicon's IP?"
date: 2025-05-25
categories: [hardware, npu]
excerpt: "A quick read of the VIP9000 spec against what Xiaomi shipped."
---

*A quick read of the VIP9000 spec against what Xiaomi shipped.*

Someone online made the claim in the title, so out of curiosity I looked at the parameters of VeriSilicon's VIP9000 IP. My read — not necessarily accurate:

*This is a translation of a piece originally published on Zhihu on May 25, 2025.*

1. First, the VIP9000 architecture is optimized for traditional mobile neural networks like CNNs and LSTMs, whereas Xiaomi's in-house NPU is clearly optimized for on-device LLM architectures. That class of computation depends more on high-bandwidth cache and highly concurrent memory access, hence the 10 MB oversized cache that the original IP doesn't have.

2. On throughput, it's evolved from the original IP's 20 TOPS to 44 TOPS. Beyond the clock-frequency gain from 3 nm, that technically isn't achievable just by adding compute units, because today's applications have many memory-bound operators (especially on-device LLMs). It needs a lot of software and hardware optimization.

3. Likewise, a lot of time on the 3 nm NPU's middle and back end.

So my conclusion: if the report is true, it should count as a fairly large modification on top of that IP. That's common in engineering — like when we write code starting from an existing base.

Whether that counts as "in-house" is a matter of taste; purely a technical discussion, additions and debate welcome.
