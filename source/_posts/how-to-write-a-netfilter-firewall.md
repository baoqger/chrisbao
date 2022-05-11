---
title: "Write a Linux firewall from scratch based on Netfilter"
date: 2022-05-04 18:06:50
tags: Linux module, Netfilter, firewall 
---

### Background

`Firewalls` are an important tool that can be configured to protect your servers and infrastructure. Firewalls' main functionalities are filtering data, redirecting traffic, and protecting against network attacks. There are both hardware-based firewalls and software-based firewalls. I will not discuss too much about firewalls, since you can find many online documents about [it](https://en.wikipedia.org/wiki/Firewall_(computing)). I 

Have you ever thought of implementing a simple firewall from scrtach? Sounds crazy? But with the power of Linux, you can do that. After you read this article, you will find that actually, it is quite simple. 

You may once use various firewalls on Linux such as [iptables](https://en.wikipedia.org/wiki/Iptables), [nftables](https://en.wikipedia.org/wiki/Nftables), [UFW](https://en.wikipedia.org/wiki/Uncomplicated_Firewall), etc. All of these firewall tools are user-space utility programs, and they are all relying on [`Netfilter`](https://en.wikipedia.org/wiki/Netfilter). `Netfilter` is Linux kernel subsystem that allows various networking-related operations to be implemented. `Netfilter` allows you to develop your firewall using the `Linux Kernel Module`.  If you don't know the techniques such as the Linux Kernel module and Netfilter, don't worry. In this article, let's write a Linux firewall from scratch based on Netfilter. You can learn the following interesting points:

- Linux kernel module development.
- Linux kernel network programming. 
- Netfilter module development.

### Netfilter and Kernel modules

### Basic guide to Kernel modules 
how to write a hello-world module
how to build it
how to debug it with advanced Linux command

### miniFirewall based on Netfilter module

#### Netfilter hook design

#### version1: just logging and accepting the packet
#### version2: filter packets based on protocol and IP address
#### version3: pass filter rules from user space to modules

1 netfilter vs bpf
  netfiltr in year 1998. bpf in year 1997
  firewall只是netfilter的一个功能而已。

2 linux module 101
  makefile 解释 
    kbuild
    -C option
    obj-m vs obj-y
  相关commnds的解释 => lsmod, insmod, rmmod, modprobe, dmesg

3 netfilter 架构解释 
  hooks
  hook function相关参数解释：priority等
  kernel source code看hook的流程 => ip_rcv

4 实现firewall
  基于protocol，基于ip address => hardcode filter条件
  config filters manually => module和user space的通信
  一个hook多个callback => 

把自己实现的功能对应于某个iptables的command，比如下面这样
iptables -A OUTPUT -p tcp -d bigmart.com -j ACCEPT
iptables -A OUTPUT -p tcp -d bigmart-data.com -j ACCEPT
iptables -A OUTPUT -p tcp -d ubuntu.com -j ACCEPT
iptables -A OUTPUT -p tcp -d ca.archive.ubuntu.com -j ACCEPT
iptables -A OUTPUT -p tcp --dport 80 -j DROP
iptables -A OUTPUT -p tcp --dport 443 -j DROP
iptables -A INPUT -p tcp -s 10.0.3.1 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -s 0.0.0.0/0 --dport 22 -j DROP

background
防火linux墙自研
Firewalls are an important tool that can be configured to protect your servers and infrastructure. 
Netfilter is a framework provided by the Linux kernel that allows various networking-related operations to be implemented including firewall.
能学到的东西:
Linux kernel module development.
Linux kernel network programming. 
Linux kernel Netfilter module development.



Netfilter is a framework provided by the Linux kernel that allows various networking-related operations to be implemented in the form of customized handlers.

Netfilter provides hooks at the critical places on the packet traversal path inside the Linux kernel. These hooks allow packet to go through additional program logics that are installed by system administrators. These program logics need to be installed inside the kernel, and the loadable kernel module technology makes it convenient to achieve that. 

Iptables is a user-space utility prograsm that allows a system administrator to configure the IP packet fitler rules of the Linux kernel firewall, implemented as different Netfilter modules. Iptables superseded ipchains; and the successor of iptables is nftables.



