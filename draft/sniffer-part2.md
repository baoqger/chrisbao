---
title: "Write a Linux packet sniffer from scratch: part two - packet filter"
date: 2022-03-17 14:25:37
tags:
---

几个问题：
1. BPF的历史: BSD -> Linux BPF -> Linux eBPF. 最后简单说一下eBPF的功能 (performance analysis and troubleshooting, js之于浏览器)。

2. BPF的作用位置 -> 性能问题 -> kernel中
before clone. source code verify

3. 从user mode注入kernel model的方法
setsockopt system call -> kernel code
sniffer implemention code(show filter code)

4. 解释filter code. BPF asm-like instruction的简单理解。这个通过很多文章都能理解了。指令里的地址就是packet的字节。

5. 如何理解in kernel virtual machine/engine这个概念。
核心在这个函数sk_run_filter. state machine



In 1992, Steven McCanne and Van Jacobson from Lawrence Berkeley Laboratory proposed a solution for BSD Unix systems for minimizing unwanted network packet copies to user space by implementing an in-kernel packet filter known as Berkeley Packet Filter (BPF). In 1997, it was introduced in Linux kernel version 2.1.75.



这里解释清楚如何使用就行了(1.从user mode向kernel model发了一段程序，这段程序会作用在每一个packet上。为什么这样做呢？因为灵活，避免了hardcode的各种rule。2通过tcpdump -dd生成filter code)。深入分析看得再来一篇。

资料：
https://www.linuxjournal.com/article/4659

第三, libpcap是支持各种过滤的，按照协议类型，按照ip地址等等调节过滤，这个如何实现呢？实现在哪个level呢？

The optimal solution to this problem is to put the filter as early as possible in the packet-processing chain(it starts at the network driver level and ends at the application level). The linux kernel allows us to put a filter, called an LPF, directly inside the PF_PACKET protocol processing routines, which are run shortly after the network card reception interrupt has been served. The filter decides which packets shall be relayed to the application and which ones should be discarded. 

In order to be as flexible as possible, and not to limit the programmer to a set of predefined conditions, the packet-filtering engine is actually implemented as a state machine running a user-defined program. The program is written in a specific pseudo-machine code language called BPF (for Berkeley packet filter), inspried by an old paper. BPF actually looks like a real assembly language with a couple of registers and a few instructions to load and store values, perform arithmetic operations and conditionally branch. 

the filter code is run on each packet to be examined, and the memory space into which the BPF processor operates are the bytes containing the packet data. The result of the filter is an integer number that specifies how many bytes of the packet the socket should pass to the application level. 

filter(LPF)可以BPF来描述，BPF本身是一个类似汇编语言一样的东西，是纯kernel层的东西。

实现的方法是：把filter写入`sock_filter`结构，然后传给`setsockopt`.

libpcap做的事情是支持将用户以string定义的filter转化为符合BPF标准的描述。

Libpcap library is an OS-independent wrapper for the BPF engine. When used on Linux machines, BPF functions are carried out by the Linux packet filter. 

One of the most useful functions provided by the libpcap is pcap_compile(), which takes a string containing a logic expression as input and output the BPF filter code. 


