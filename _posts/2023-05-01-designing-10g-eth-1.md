---
layout: post
title:  "Designing a Low Latency 10G Ethernet Core - Part 1 (Introduction)"
date:   2023-05-01 15:43:00 +0100
categories: ethernet
---

[![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ttchisholm/10g-low-latency-ethernet) View the code on GitHub 

*Links to the other parts in this series:*
1. [Introduction]({% link _posts/2023-05-01-designing-10g-eth-1.md %}) *(this post)*
2. [Design Overview and Verification]({% link _posts/2023-05-02-designing-10g-eth-2.md %}) 
3. [Low Latency Techniques]({% link _posts/2023-05-06-designing-10g-eth-3.md %})
4. [Performance Measurement and Comparison]({% link _posts/2023-05-07-designing-10g-eth-4.md %})
5. [Potential Improvements]({% link _posts/2023-05-07-designing-10g-eth-5.md %})

## Introduction

This is the first in a series of blog posts describing my experience developing a low latency 10G Ethernet core for FPGA. I decided to do this as a personal project to develop expertise in low latency FPGA design and high-speed Ethernet, as well as to experiment with tools and techniques that I could use full-time. As a small spoiler, the design has less than **60ns loopback latency**, which is comparable to commercial offerings.

These posts will focus on the things that are likely different to a 'standard' design, as I believe this will be more interesting to the reader. Specifically:
- The use of [cocotb](https://www.cocotb.org/) and [pyuvm](https://github.com/pyuvm/pyuvm) for verification
- The techniques implemented to reduce packet processing latency
- Analysis of commercially available low latency and 'ultra'-low latency cores
- Latency measurement results and comparison
- Other techniques not implemented

If the reader is unfamiliar with Layer 1/2 Ethernet, I would recommend the following resources:
- [10G Ethernet Layer 1 Overview](https://old.fmad.io/blog-10g-ethernet-layer1-overview.html)
- [YouTube - The Big MAC Mystery](https://www.youtube.com/watch?v=4YRsq8iM4n0)
- [IEEE Standard for Ethernet](https://standards.ieee.org/ieee/802.3/10422/) - Full Ethernet (802.3) spec
- [64B/66B overview](https://www.ieee802.org/3/10GEPON_study/public/july06/thaler_1_0706.pdf) - Overview of 10G PCS from the spec above

Next Post - [Design Overview and Verification]({% link _posts/2023-05-02-designing-10g-eth-2.md %}) 

