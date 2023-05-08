---
layout: post
title:  "Designing a Low Latency 10G Ethernet Core - Part 5 (Potential Improvements)"
date:   2023-05-07 14:58:00 +0100
categories: ethernet
---

[![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ttchisholm/10g-low-latency-ethernet) View the code on GitHub 

*Links to the other parts in this series:*
1. [Introduction]({% link _posts/2023-05-01-designing-10g-eth-1.md %}) 
2. [Design Overview and Verification]({% link _posts/2023-05-02-designing-10g-eth-2.md %}) 
3. [Low Latency Techniques]({% link _posts/2023-05-06-designing-10g-eth-3.md %}) 
4. [Performance Measurement and Comparison]({% link _posts/2023-05-07-designing-10g-eth-4.md %})
5. [Potential Improvements]({% link _posts/2023-05-07-designing-10g-eth-5.md %}) *(this post)*

## Potential Improvements

In this final part, I'll discuss potential ways the design can be improved to reduce latency and compete with the best on the market.

1. 16-bit, ~644MHz
2. Tighter MAC/PCS integration
4. Clock Synchronisation

### 16-bit, ~644MHz

There are three parts to this. Firstly, running the GTY transceivers with a 16-bit internal datapath is shown to reduce latency by **~13ns** ([Low Latency Transceiver Designs for Quantitative Finance](https://www.xilinx.com/developer/articles/low-latency-transceiver-designs-for-fintech.html)). It is possible to do this with a 32-bit user interface but unfortunately it is not possible to configure this in the gtwizard IP and I believe it may require a -3 speed grade device to pass timing. 

Secondly, running the whole design at ~644MHz with a 16-bit interface will reduce total latency by an additional **3.1ns**. This is because the first 16-bits of data can be presented a half-cycle earlier when compared with a ~322MHz design.

Finally, in our test setup where the start of receive packet trigger is crossed into the TX domain, when running at ~644MHz this crossing takes half the time, saving **3.1ns**.

**Potential latency improvement = ~19.2ns**

### Tighter MAC/PCS integration

There are currently a number of opportunities to improve the latency by further integrating the MAC/PCS. For example, currently when user data is presented to the TX MAC, this is buffered for two cycles while the start of frame and preamble is sent. It may be possible to 'pre-load' the PCS and CRC calculation with this data and use a flag to switch to it instead of the idle frames.

Additionally on transmit, currently as there is no way to know how many bytes are in the packet presented to the TX MAC, all 8 options for the TERM frame type are supported. This means that the encoder in the PCS must wait for all 64-bits of data in the frame before progressing, as the block type is transmitted first. By padding packet sizes to multiples of 8 bytes, only one TERM frame type would need to be supported and the PCS would therefore not have to buffer this data.

Finally, removing the conversion to XGMII between the MAC/PCS may reduce the logic in the datapath, which could help to achieve ~644MHz without additional pipelining.

**Potential latency improvement = 6.2 to 12.4ns**

### Clock Synchronisation

The two improvements above brings us close to the ~20ns target set by the LDA Tech and Orthogone cores. However, implementing TX/RX clock synchronisation would have a big impact on real-world latency by removing the need for clock domain crossing between TX/RX domains. This could be achieved by using the recovered clock from the RX transceiver to generate the TX transceiver clock, rather than using an on-board reference. This approach is used in [Synchronous Ethernet](https://en.wikipedia.org/wiki/Synchronous_Ethernet) and [White Rabbit](https://en.wikipedia.org/wiki/White_Rabbit_Project) time transfer protocols. 

Unfortunately, this process is not trivial, as recovered clocks tend to have much higher phase noise (jitter) than stable external references for high speed communications. Therefore the clock needs to be 'cleaned' using a special class of PLLs referred to as jitter attenuators, such as the [Si5395](https://www.skyworksinc.com/-/media/Skyworks/SL/documents/public/data-sheets/si5395-94-92-a-datasheet.pdf) found on the [LDA Orion Series](https://ldatech.com/Boards/LDA_Orion).

However, Xilinx Ultrascale+ devices can perform clock recovery using FRACXO as described in [All Digital VCXO Replacement Using a Gigabit Transceiver Fractional PLL Application Note (XAPP1276)](https://docs.xilinx.com/v/u/en-US/xapp1276-vcxo). I would presume the phase noise of FRACXO method to be worse than an external, specialised chip, so more investigation would be required here.

