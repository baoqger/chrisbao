---
title: "Write a Linux firewall from scratch based on Netfilter: part one- Netfilter and Kernel Modules"
date: 2022-05-04 18:06:50
tags: Linux module, Netfilter, firewall 
---

### Background

`Firewalls` are an important tool that can be configured to protect your servers and infrastructure. Firewalls' main functionalities are filtering data, redirecting traffic, and protecting against network attacks. There are both hardware-based firewalls and software-based firewalls. I will not discuss too much about the background here, since you can find many online documents about [it](https://en.wikipedia.org/wiki/Firewall_(computing)).

Have you ever thought of implementing a simple firewall from scratch? Sounds crazy? But with the power of Linux, you can do that. After you read this series of articles, you will find that actually, it is quite simple. 

You may once use various firewalls on Linux such as [iptables](https://en.wikipedia.org/wiki/Iptables), [nftables](https://en.wikipedia.org/wiki/Nftables), [UFW](https://en.wikipedia.org/wiki/Uncomplicated_Firewall), etc. All of these firewall tools are user-space utility programs, and they are all relying on [`Netfilter`](https://en.wikipedia.org/wiki/Netfilter). `Netfilter` is the Linux kernel subsystem that allows various networking-related operations to be implemented. `Netfilter` allows you to develop your firewall using the `Linux Kernel Module`.  If you don't know the techniques such as the Linux Kernel module and Netfilter, don't worry. In this article, let's write a Linux firewall from scratch based on Netfilter. You can learn the following interesting points:

- Linux kernel module development.
- Linux kernel network programming. 
- Netfilter module development.

### Netfilter and Kernel modules
#### Basics of Netfilter
`Netfilter` can be considered to be the third generation of `firewall` on Linux. Before `Netfilter`was introduced in Linux Kernel 2.4, there are two older generations of firewalls on Linux as follows: 
  - The first generation was a port of an early version of BSD UNIX's `ipfw` to Linux 1.1. 
  - The second generation was `ipchains` developed in the 2.2 series of Linux Kernel. 

As we mentioned above, `Netfilter` was designed to provide the infrastructure inside the Linux kernel for various networking operations. So `firewall` is just one of the multiple functionalities provided by `Netfilter` as follows:

<img src="/images/netfilter-arch.png" title="Netfilter architecture" width="600px" height="400px">

 - **Packet filtering**: is in charge of filtering the packets based on the rules. It is also the topic of this article. 
 - **NAT (Network address translation)**: is in charge of translating the IP address of network packets. `NAT` is an important protocol, which has become a popular and essential tool in `conserving global address space in the face of IPv4 address exhaustion`. If you don't know `NAT` protocol, you can refer to other [documents](https://en.wikipedia.org/wiki/Network_address_translation). I will examine it in other future articles. 
 - **Packet mangling**: is in charge of modifying the packet content(In fact, `NAT` is one kind of packet mangling, which modifies the source or destination IP address). For example, `MSS (Maximum Segment Size)` value of TCP SYN packets can be altered to allow large-size packets transported over the network. 

Note: this article will focus on building a simple firewall to filter packets based on Netfilter. So the `NAT` and `Packet Mangling` parts are not in the scope of this article. 

Packet filtering can only be done inside the Linux kernel (Netfilter's code is in the kernel as well), if we want to write a mini firewall, it has to run in the kernel space. Right? Does it mean we need to add our code into the kernel and recompile the kernel? Imagine you have to recompile the kernel each time you want to add a new packet filtering rule. That's a bad idea. The good news is that `Netfilter` allows you to add extensions using the [`Linux kernel modules`](https://wiki.archlinux.org/title/Kernel_module). 
#### Basics of Linux Kernel modules

Although Linux is a [`monolithic kernel`](https://en.wikipedia.org/wiki/Monolithic_kernel), it can be extended using kernel modules. Modules can be inserted into the kernel and removed on demand. Linux isolates the kernel but allows you to add specific functionality on the fly through modules. In this way, Linux keeps a balance between stability and usability. 

I want to examine one confusing point about the kernel module here: what is the difference between `driver` and `module`:

 - A driver is a bit of code that runs in the kernel to talk to some hardware device. It drives the hardware. Standard practice is to build drivers as kernel modules where possible, rather than link them statically to the kernel since that gives more flexibility. 
 - A kernel module may not be a device driver at all.    

### Summary

In the first post of this series, we examine the basics of Netfilter and Linux kernel modules. In the next post, let's start implementing the mini firewall. 


