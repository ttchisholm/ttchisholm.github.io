---
layout: post
title:  "Designing a Low Latency 10G Ethernet Core - Part 4 (Performance Measurement and Comparison)"
date:   2023-05-07 12:00:00 +0100
categories: ethernet
---

[![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ttchisholm/10g-low-latency-ethernet) View the code on GitHub 

*Links to the other parts in this series:*
1. [Introduction]({% link _posts/2023-05-01-designing-10g-eth-1.md %}) 
2. [Design Overview and Verification]({% link _posts/2023-05-02-designing-10g-eth-2.md %}) 
3. [Low Latency Techniques]({% link _posts/2023-05-06-designing-10g-eth-3.md %})
4. [Performance Measurement and Comparison]({% link _posts/2023-05-07-designing-10g-eth-4.md %}) *(this post)*
5. [Potential Improvements]({% link _posts/2023-05-07-designing-10g-eth-5.md %})

## Performance Measurement and Comparison

In this part, we will analyse the performance of the core and compare it to commercial offerings. 

### Market Analysis

There are a number of low latency and ultra-low latency IP cores available commercially, primarily from [HiTek Systems](https://hiteksys.com/), [Orthogone](https://orthogone.com/) and [LDA Technologies](https://ldatech.com/). Each provide a product brief for their cores which I have summarised below.


| Name                                                                       | Estimated Loopback Latency (ns)      | Measurement Method                                                          | Interface Width | Core Clock | Notes |
|----------------------------------------------------------------------------|--------------------------------------|----------------------------------------------------------------------------|-----------------|------------|-------|
| [HiTek Systems 10G Low Latency Ethernet FPGA IP Core Solution](https://hiteksys.com/fpga-ip-cores/10g-low-latency-ethernet) | 318.8 + transceiver latency (**346.63ns**) | Sum of MAC, PCS delay for Tx and Rx | 32-bit | | Supports older (6-series) parts. Designed in 2012 |
| [HiTek Systems Ultra-Low Latency 10G Ethernet IP Solution](https://hiteksys.com/pdf/10G-Ethernet-Ultra-Low Latency-Product-Brief.pdf) | 52.7 + transceiver latency (**80.53ns**) | Sum of MAC, PCS delay for Tx and Rx | 32-bit  |  | |
| [Orthogone Ultra-Low Latency 10G Ethernet MAC and PCS (Previous generation)](https://orthogone.ca/wp-content/uploads/OT-Ultra-Low Latency-10G-Ethernet-MAC-and-PCS-pb.pdf) | **67ns** | Sum of MAC, PCS, GTY delay for Tx and Rx  | 32-bit | ~322MHz |  |
| [Exablaze ExaMAC](https://www.youtube.com/watch?v=1zGXAAthKqY) | **45.72ns** | TxSoP to RxSoP with CDC, minus cable delay | 32-bit | ~322MHz | Exablaze now part of Cisco, core seemingly discontinued |
| [Orthogone Ultra-Low Latency 10G Ethernet MAC and PCS (Current Generation)](https://info.orthogone.com/hubfs/PDF/OT-Ultra-Low%20Latency%2010G%20Ethernet%20MAC%20and%20PCS-pb.pdf) | **34.1ns** | "GTY + MAC/PCS + CDC (Measured from TxSoP to RxSoP using serial loopback)" | 32-bit          | ~644MHz    | |
| [LDA Tech Ultra-low latency 10 GbE MAC/PCS (32bit/322MHz)](https://ldatech.com/Solutions/Ultra_Low_Latency) | 19.9 (MAC/PCS) + 15.6 (GTY) = **33.5ns**   | Sum of MAC, PCS, GTY delay for Tx and Rx + Rx CDC | 32-bit | ~322MHz | Note the 16-bit GTY delay quoted. If 32-bit mode used in GTY total latency is **47.43ns** |
| [LDA Tech Ultra-low latency 10 GbE MAC/PCS (16bit/644MHz)](https://ldatech.com/Solutions/Ultra_Low_Latency)                   | 6.2 (MAC/PCS) + 15.6 (GTY) = **21.8ns** | Sum of MAC, PCS, GTY delay for Tx and Rx | 16-bit | ~644MHz | |
| [Orthogone Ultra-Low Latency 10G Ethernet MAC and PCS (Current Generation)](https://info.orthogone.com/hubfs/PDF/OT-Ultra-Low%20Latency%2010G%20Ethernet%20MAC%20and%20PCS-pb.pdf)  | **20.2ns** | “GTY + MAC/PCS (Measured from TxSoP to RxSoF using serial loopback)” | 16-bit | ~644MHz | Note difference between SoP (Start of Packet) and SoF (Start of Frame) |

There's a range of performances here, but also a range of measurement methods which is important to keep in mind when comparing cores. Note the HiTek cores don't provide the transceiver latency values, so I've obtained them from this analysis from Xilinx - [Low Latency Transceiver Designs for Quantitative Finance](https://www.xilinx.com/developer/articles/low-latency-transceiver-designs-for-fintech.html).

## Performance Measurement

As seen above, there are a number of ways to measure the latency of the core. An excellent analysis of these methods is presented by Exablaze here - [The Big Mac Mystery - How do you measure a MAC?](https://www.youtube.com/watch?v=1zGXAAthKqY&) and summarised below:

1. Count the cycles taken for data to propagate through MAC/PCS for transmit and receive
2. Connect transceivers in loopback, measuring time taken from the start of transmit packet to start of receive packet (crossed into TX clock domain)
3. Connect MAC interfaces in loopback, measuring time taken from the start of receive packet to start of transmit packet at the transceiver interface
4. Set a trigger to send a packet when the eSoF (PCS start of frame) is detected, measuring time taken from the start of receive packet to start of transmit packet
5. Set a trigger to send a packet when the SoF (MAC start of frame) is detected, measuring time taken from the start of receive packet to start of transmit packet

Option 1 is the most commonly quoted on the product briefs, but is also the least realistic. Options 3,4 and 5 require specialist timing equipment which I don't have access to, and seemingly none of the other cores listed are tested in this manner. Option 2 presented the best balance between practicality and a representative test scenario. 

As discussed in [Part 2]({% link _posts/2023-05-02-designing-10g-eth-2.md %}), the example design measures latency by connecting the transceiver in serial loopback. A signal is sent to a timer a the start of a transmit packet, which counts until the start of the receive packet is detected, after crossing the clock domains.

![Example Design Overview](/assets/images/designing-10g-eth/example.png)
<p style="text-align: center;"><b>Figure 1: Example Design Overview</b></p>

It's worth noting there are arguably multiple options for detecting the 'start of a packet':

1. The first byte of user data (payload) presented by the MAC
2. The MAC layer start of frame delimiter (SFD = 0xD5)
3. The PCS layer start character (RS START = 0xFB). This replaces the first octet of the 7 octet preamble.

Exablaze use the terms eSoF (early start of frame) and SoF (start of frame) to distinguish these which I will also use. However, Orthogone reference SoP and SoF, and it's not clear how these align. 

![Start of Layer 2 Ethernet Frame](/assets/images/designing-10g-eth/start-frame.png)
<p style="text-align: center;"><b>Figure 2: Start of Layer 2 Ethernet Frame (prior to block encoding)</b></p>

For my testing, the start trigger is the cycle on which data is presented to the core and the core is ready to receive it, and the end trigger is when the first byte of user data in the payload is received. This is the most pessimistic option, however the usefulness of the SoF character without any user data is arguable, so I feel that it is the most representative. 

The measured average latency for the core using the method above is **58.2ns**. This was measured across a few thousand packets of different sizes using a script to automatically capture data from the latency measurement ILA in the example design. 

![ILA Latency](/assets/images/designing-10g-eth/latency_ila.png)
<p style="text-align: center;"><b>Figure 3: Latency Measurement ILA</b></p>

As the SoF arrives exactly 1 cycle earlier, and eSoF arrives 2 cycles earlier, we can calculate alternative latency values for comparison:

1. User data TX to User data RX (including CDC): **58.2ns**
2. User data TX to SoF (SFD) (including CDC): **55.1ns**
3. User data TX to eSoF (RS START) (including CDC): **52.0ns**

Regardless of methodology, this puts us right in the middle of the cores analysed above, but at the lower end of performance for the latest designs. In the next and final part, I'll look at ways the design could be improved to compete with the best on the market.

Next Post - [Potential Improvements]({% link _posts/2023-05-07-designing-10g-eth-5.md %})