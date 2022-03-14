---
title: Write a Linux packet sniffer from scratch
date: 2022-02-22 10:21:14
tags: packet sniffer, socket, PF_PACKET
---

### Background

When we refer to network packet sniffer, some famous and popular tools like: `tcpdump` come to your mind . In my previous articles, I once shown you how to use it to capture network packet. But have you ever think about writing a packet sniffer from scratch? In this article, let's do it. After reading this article, you will find that it is not as difficult as you may think. 

### Introduction

First, let's review how `tcpdump` is implemented. As a powerful command-line tool, it is built on top of the library `libpcap`. So the power of `tcpdump` is original from `libpcap`, which is based on the great research result from Berkeley, in detail you can refer to [this paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf).

As you know, different operating systems have different internal implementations about network stacks. `libpcap` covers all of these differences and provides system-independent interface for user-level packet capture. In my article, I want to focus on `Linux` platform, so how `libpcap` works on Linux system? According to some [documents](https://stackoverflow.com/questions/21200009/capturing-performance-with-pcap-vs-raw-socket), it turns out that libpcap uses the `PF_PACKET` socket to capture packets on an network interface.

So the next question is what is `PF_PACKET` socket? 

### PF_PACKET socket

In my previous [article](https://organicprogrammer.com/2021/07/31/how-to-implement-simple-http-server-golang/), we mentioned that the socket interface is really TCP/IP’s window on the world. In most modern systems incorporating TCP/IP, the socket interface is the only way that applications make use of the TCP/IP suite of protocols. 

It is correct. This time, let's dig deeper about `socket` by examining the system call executed when we create a new socket: 

<img src="/images/pf-packet-socket.png" title="PF_PACKET socket" width="400px" height="300px">


```c
int socket(int domain, int type, int protocol);
```

When you want to create a socket with the above system call, you have to specify which domain (or protocol family) you are going to use with that socket as the first argument. The most commonly used family is `PF_INET`, which is for communications based on IPv4 protocols (when you create a TCP server, you use this family). Moreover, you have to specify a type for your socket as the second argument. And the possible values depend on the family you specified. For example, when dealing with the `PF_INET` family, the values for type include `SOCK_STREAM`(for TCP) and `SOCK_DGRAM`(for UDP). For other detail information about socket system call, you can refer to socket(3) man page. 

You can find one potential value for `domain` argument as follows:

```
 AF_PACKET    Low-level packet interface
```
**Note**: `AF_PACKET` and `PF_PACKET` is the same thing. In the history, it is firstly called `PF_PACKET` and then renamed to `AF_PACKET`. `PF` means protocol families and `AF` means address families. In this article, we'll use `PF_PACKET`. 

Different from `PF_INET` socket, which can give you TCP segment. By `PF_PACKET` socket, you can get raw `Ethernet` frame which bypasses the usual upper layer handling of TCP/IP stack. It might sound a little bit crazy, right? But, that is, any packet received will be directly passed to the application. 

Just to understand `PF_INET` socket better, let us go deeper and roughly examine the path of a received packet from netowrk interface to the application level. 

As shown in the image above, when the network interface card(NIC) receive a packet, it will be handled by the driver. The driver maintains a strcuture called `ring buffer` and write the packet to kernel memory (the memory is pre-allocated with ring buffer)  with direct memory access(DMA). The packet is stored inside a structure called **`sk_buff`**(one of the most important structures related to kernel network subsystem).   

After entering the kernel space, the packet goes through protocol stack handling layer by layer, such as `IP processing`and `TCP/UDP processing`. And then the packet goes into applications via socket interface. You already understand this familiar path very well.

But for `PF_PACKET` socket, the packet in `sk_buff` will be cloned, then it skips the protocol stacks and directly goes to the application. **Note** the clone operation is needed, because one copy is consumed by the `PF_PACKET` socket, and the other one goes through the usual protocol stacks.

思路：

背景：libpcap是一个很强大的库，它是如何实现的呢？如何在linux下实现类似的功能呢？

内容：

1 packet socket的介绍。socket是kernel networking stack对application开放的连接点。把连接点开在transport layer上，就是TCP socket, 通过它application可以得到TCP segment。类似的如果把连接点开放在ethernet layer上，就是Packet socket, 通过它application可以到ethernet frame.

上面这种理解可以通过 socket()系统调用的参数验证。

稍微写一点 path of a packet in the linux kernel stack, 今后再扩展.



2 bind to device
network device interface list,
ifconfig output输出分析
loopback大致介绍

3 promiscuous mode
hub和switch区别介绍

4 Linux packet filter. 
这里解释清楚如何使用就行了(1.从user mode向kernel model发了一段程序，这段程序会作用在每一个packet上。为什么这样做呢？因为灵活，避免了hardcode的各种rule。2通过tcpdump -dd生成filter code)。深入分析看得再来一篇。

资料：
https://www.linuxjournal.com/article/4659

libpcap这个library实现了跨平台的packet capture, 但是我们只关注linux. 如何在linux 系统下实现跟libpcap一样的（或者类似的）packet capture功能呢？

第一，libpcap的实现是基于AF_PACKET这个类型的socket的。这也是我们的简单实现的基础: AF_PACKET socket. 

In recent versions of the Linux kernel(post-2.0 release), a new protocol family has been introduced, named PF_PACKET. This family allows an application to send and receive packets dealing directly with the network card driver, thus avoiding the usual protocol stack-handling. That is, any packet sent through the socket will be directly passed to the Ethernet interface, and any packet received through the interface will be directly passed to the application. 

第二，libpcap有一个功能是：promiscuous mode。这个是如何实现的呢？

The PF_PACKET family allows an application to retrieve data packets as they are received at the network card level, but still does not allow it to read packets that are not addressed to its host. As we have seen before, this is due to the network card discarding all the packets that do not contain its own MAC address - an operation mode called non-promiscuous, which basically means that each network card is minding tis own business and reading only the frames directed to it. 

We need to enable a network card to be set in promiscuous mode to pick up all the packets it sees. 

To set a network card to promiscuous mode, all we have to do is issue a particular `ioctl()` system call to an open socket on that card. 

第三, libpcap是支持各种过滤的，按照协议类型，按照ip地址等等调节过滤，这个如何实现呢？实现在哪个level呢？

The optimal solution to this problem is to put the filter as early as possible in the packet-processing chain(it starts at the network driver level and ends at the application level). The linux kernel allows us to put a filter, called an LPF, directly inside the PF_PACKET protocol processing routines, which are run shortly after the network card reception interrupt has been served. The filter decides which packets shall be relayed to the application and which ones should be discarded. 

In order to be as flexible as possible, and not to limit the programmer to a set of predefined conditions, the packet-filtering engine is actually implemented as a state machine running a user-defined program. The program is written in a specific pseudo-machine code language called BPF (for Berkeley packet filter), inspried by an old paper. BPF actually looks like a real assembly language with a couple of registers and a few instructions to load and store values, perform arithmetic operations and conditionally branch. 

the filter code is run on each packet to be examined, and the memory space into which the BPF processor operates are the bytes containing the packet data. The result of the filter is an integer number that specifies how many bytes of the packet the socket should pass to the application level. 

filter(LPF)可以BPF来描述，BPF本身是一个类似汇编语言一样的东西，是纯kernel层的东西。

实现的方法是：把filter写入`sock_filter`结构，然后传给`setsockopt`.

libpcap做的事情是支持将用户以string定义的filter转化为符合BPF标准的描述。

Libpcap library is an OS-independent wrapper for the BPF engine. When used on Linux machines, BPF functions are carried out by the Linux packet filter. 

One of the most useful functions provided by the libpcap is pcap_compile(), which takes a string containing a logic expression as input and output the BPF filter code. 
