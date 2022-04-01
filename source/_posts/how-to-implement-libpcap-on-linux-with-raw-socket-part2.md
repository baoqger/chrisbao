---
title: "Write a Linux packet sniffer from scratch: part two- BPF"
date: 2022-03-28 14:15:15
tags: BPF, JIT, virtual machine
---

### Introduction

In the [previous article](https://organicprogrammer.com/2022/02/22/how-to-implement-libpcap-on-linux-with-raw-socket-part1/), we examined how to develop a network sniffer with `PF_SOCKET` socket in Linux platform. The sniffer developed in the last article captures all the network packets. But a powerful network sniffer like `tcpdump` should provide the packet filtering functionality. For instance, the sniffer can only capture the `TCP` segment(and skip the UPD), or it can only capture the packets from a specific source IP address. In this article, let's continue to explore how to do that. 

### Background of BPF

`Berkeley Packet Filter(BPF)` is the essential underlying technology for packet capture in Unix-like operating systems.  
Search BPF as the keyword online, and the result is very confusing. It turns out that `BPF` keeps evolving, and there are several associated concepts such as `BPF` `cBPF` `eBPF` and `LSF`. So let us examine those concepts along the timeline:

- In **1992**, `BPF` was first introduced to the BSD Unix system for filtering unwanted network packets. The proposal of BPF was from researchers in Lawrence Berkeley Laboratory, who also developed the `libpcap` and `tcpdump`. 

- In **1997**, Linux Socket Filter(LSF) was developed based on BPF and introduced in Linux kernel version 2.1.75. Note that `LSF` and `BPF` have some distinct differences, but in the Linux context, when we speak of BPF or LSF, we mean the same packet filtering mechanism in the Linux kernel. We'll examine the detailed theory and design of BPF in the following sections. 

- Originally, BPF was designed as a network packet filter. But in **2013**, BPF was widely extended, and it can be used for non-networking purposes such as performance analysis and troubleshooting. Nowadays, the extended BPF is called `eBPF`, and the original and obsolete version is renamed to classic BPF (`cBPF`). **Note that what we examine in this article is cBPF, and eBPF is not inside the scope of this article**. `eBPF` is the hottest technology in today's software world, and I'll talk about it in the future. 

### Where to place BPF

The first question to answer is where should we place the filter. The last article examines the path of a received packet  as follows: 

<img src="/images/pf-packet-socket.png" title="PF_PACKET socket" width="400px" height="300px">

The best solution to this question is to put the filter as early as possible in the path. Since copying a large amount of data from kernel space to the user space produces a huge overhead, which can influence the system performance a lot. So BPF is a kernel feature. The filter should be triggered immediately when a packet is received at the network interface. Let's verify this behavior by examining the kernel source code as follows (**Note** the kernel code shown in this article is based on version 2.6, which contains the `cBPF` implementation.): 

```c
/* source code file of net/packet/af_packet.c */
/* packet_create: create socket */
static int packet_create(struct net *net, struct socket *sock, int protocol)
{
    /* some code omitted ... */
	po = pkt_sk(sk);
	sk->sk_family = PF_PACKET;
	po->num = proto;

	spin_lock_init(&po->bind_lock);
	po->prot_hook.func = packet_rcv; // attach hook function to socket

	if (sock->type == SOCK_PACKET)
		po->prot_hook.func = packet_rcv_spkt; // attach hook function to socket

	if (proto) {
		po->prot_hook.type = proto;
		dev_add_pack(&po->prot_hook);
		sock_hold(sk);
		po->running = 1;
	}
}
```
`packet_create` function handles the socket creation when the application calls the `socket` system call. In lines 11 and 14, it attaches the hook function to the socket. The hook function executes when the packet is received.

The following code block shows the hook function `packet_rcv`:  

```c
/* hook function packet_rcv is triggered, when the packet is received */
static int packet_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    /* some code omitted ... */
    sk = pt->af_packet_priv;
    snaplen = skb->len;
    res = run_filter(skb, sk, snaplen); // filter logic
    if (!res)
	    goto drop_n_restore; // drop the packet

    __skb_queue_tail(&sk->sk_receive_queue, skb); // put the packet into the queue
}
```
`packet_rcv` function calls `run_filter`, which is just the BPF logic part(Currently, you can regard it as a black box. In the next section, we'll examine the details). Based on the return value of `run_filter` the packet can be filtered out or put into the queue. 

So far, you can understand BPF(or the packet filtering) is working inside kernel space. But the packet sniffer is a user-space application. The next question is how to link the filtering rules in user space to the filtering handler in kernel space. 

To answer this question, we have to understand BPF itself. It's right time to understand this great piece of work. 

### BPF machine

As I mentioned above, `BPF` was introduced in this original [paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf) written by the researchers from Berkeley. I strongly recommend you to read this great paper based on my own experience. At the beginning, I feel scary to read it, so I read other related documents and try to understand BPF. But most documents only cover one portion of the entire system, so it's very hard to piece all the information together. Finally I read the original paper, and connect all parts together. **As the saying goes, sometimes taking time is actually a shortcut.**

```c
static inline unsigned int run_filter(struct sk_buff *skb, struct sock *sk,
				      unsigned int res)
{
	struct sk_filter *filter;

	rcu_read_lock_bh();
	filter = rcu_dereference(sk->sk_filter);
	if (filter != NULL)
		res = sk_run_filter(skb, filter->insns, filter->len); // the filtering is inside sk_run_filter function
	rcu_read_unlock_bh();

	return res;
}
```

