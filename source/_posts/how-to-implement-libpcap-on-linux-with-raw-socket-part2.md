---
title: "Write a Linux packet sniffer from scratch: part two- BPF"
date: 2022-03-28 14:15:15
tags: BPF
---

### Introduction

In the [previous article](https://organicprogrammer.com/2022/02/22/how-to-implement-libpcap-on-linux-with-raw-socket-part1/), we examined how to develop a network sniffer with `PF_SOCKET` socket in Linux platform. The sniffer developed in the last article will capture all the network packets. But a powerful network sniffer like `tcpdump` should provide the packet filtering functionality. For instance, the sniffer can only capture `TCP` segment(and UPD will be skipped), or it can only capture the packets from a specific source IP address. In this article, let's continue to explore how to do that. 

### Background of BPF

`Berkerly Packet Filter(BPF)` is the key technology for packet capture in Unix-like operating systems.  
Search BPF as the keyword online, the result will be very confusing, especially at the time when you just start learning it. It turns out that `BPF` keeps evolving over time, there are several associated concepts such `BPF`, `cBPF`, `eBPF` and `LSF`, let us examine those concepts along the timeline:

- In **1992**, `BPF` was first introduced to BSD Unix system for filtering the unwanted network packets. BPF was proposed by researchers from Lawrence Berkeley Laboratory, who also developed the `libpcap` and `tcpdump`. 

- In **1997**, Linux Socket Filter(LSF) was developed based on BPF and introduced in Linux kernel version 2.1.75. Note that `LSF` and `BPF` have some distinct differences, but in Linux context, when we speak of BPF or LSF, we mean the same mechanism of filtering in the Linux kernel. We will examine the detailed theory and mechanism of BPF in next sections. 

- BPF is originally designed as a network packet filter, but in **2013** BPF was extended greatly and it can be used for non-networking purposes such as performance analysis and troubleshooting. Nowadays, the extended BPF is call `eBPF` and the original and obsolete version is renamed to classic BPF (`cBPF`). **Note we will examine cBPF in this article, eBPF is not inside the scope of this article.**



