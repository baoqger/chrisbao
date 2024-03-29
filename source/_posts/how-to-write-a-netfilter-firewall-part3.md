---
title: "Write a Linux firewall from scratch based on Netfilter: part three - Netfilter module"
date: 2022-06-08 14:59:19
tags: Linux module, Netfilter, firewall 
---

### Background
In the previous [article](https://organicprogrammer.com/2022/05/05/how-to-write-a-netfilter-firewall-part2/), we examined how to write a Kernel module and load it dynamically into a running Linux system. Based on this understanding, let's continue our journey to write a `Netfilter` module as our mini-firewall.  

### Netfilter architecture.

##### Basics of Netfilter hooks
**The `Netfilter` framework provides a bunch of `hooks` in the Linux kernel. As network packets pass through the protocol stack in the kernel, they will traverse these hooks as well**. And Netfilter allows you to write modules and register callback functions with these hooks. When the hooks are triggered, the callback functions will be called. This is the basic idea behind Netfilter architecture. Not difficult to understand, right? 

<img src="/images/netfilter-in-kernel.png" title="Netfilter architecture" width="800px" height="600px">

Currently, Netfilter provides the following 5 hooks for `IPv4`:
- *NF_INET_PRE_ROUTING*: is triggered right after the packet has been received on a network card. This hook is triggered before the `routing decision` was made. Then the kernel determines whether this packet is destined for the current host or not. Based on the condition, the following two hooks will be triggered. 
- *NF_INET_LOCAL_IN*: is triggered for network packets that are destined for the current host. 
- *NF_INET_FORWARD*: is triggered for network packets that should be forwarded. 
- *NF_INET_POST_ROUTING*: is triggered for network packets that have been routed and before being sent out to the network card. 
- *NF_INET_LOCAL_OUT*: is triggered for network packets generated by the processes on the current host.

The hook function you defined in the module can mangle or filter the packets, but it eventually must return a status code to Netfilter. There are several possible values for the code, but for now, you only need to understand two of them: 

- *NF_ACCEPT*: this means the hook function accepts the packet and it can go on the network stack trip. 
- *NF_DROP*: this means the packet is dropped and no further parts of the network stack will be traversed.

Netfilter allows you to register multiple callback functions to the same hook with different priorities. If the first hook function accepts the packet, then the packet will be passed to the next functions with low priority. If the packet is dropped by one callback function, then the next functions(if existing) will not be traversed. 

As you see, `Netfilter` has a big scope and I can't cover every detail in the articles. So the mini-firewall developed here will work on the hook `NF_INET_PRE_ROUTING`, which means it works by controlling the inbound network traffic. But the way of registering the hook and handling the packet can be applied to all other hooks. 

*Note*: there is another remarkable question: what's the difference between `Netfilter` and `eBPF`? If you don't know eBPF, please refer to my previous [article](https://organicprogrammer.com/2022/03/28/how-to-implement-libpcap-on-linux-with-raw-socket-part2/). Both of them are important network features in the Linux kernel. The important thing is `Netfilter` and `eBPF` hooks are located in different layers of the Kernel. As I drew in the above diagram, `eBPF` is located in a lower layer. 

##### Kernel code of Netfilter hooks

To have a clear understanding of how the `Netfilter` framework is implemented inside the protocol stack, let's dig a little bit deeper and take a look at the kernel source code (Don't worry, only shows several simple functions). Let's use the hook `NF_INET_PRE_ROUTING` as an example; since the mini-firewall will be written based on it. 

When an IPv4 packet is received, its handler function `ip_rcv` will be called as follows: 

```c
//In source code file /kernel-src/net/ipv4/ip_input.c
/*
 * IP receive entry point
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
           struct net_device *orig_dev)
{
        struct net *net = dev_net(dev);

        skb = ip_rcv_core(skb, net);
        if (skb == NULL)
                return NET_RX_DROP;
        // run Netfilter NF_INET_PRE_ROUTING hook's callback function
        return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, 
                       net, NULL, skb, dev, NULL,
                       ip_rcv_finish);
}
```
In this handler function, you can see the hook is passed to the function `NF_HOOK`. Based on the name `NF_HOOK`, you can guess that it is for triggering the Netfilter hooks. Right? Let's continue to examine how `NF_HOOK` is implemented as follows: 
```c
//In source code file /kernel-src/include/linux/netfilter.h
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
        struct net_device *in, struct net_device *out,
        int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
        int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
        if (ret == 1)
                ret = okfn(net, sk, skb); // in our case: okfn is ip_rcv_finish
        return ret;
}
/**
 *      nf_hook - call a netfilter hook
 *
 *      Returns 1 if the hook has allowed the packet to pass.  The function
 *      okfn must be invoked by the caller in this case.  Any other return
 *      value indicates the packet has been consumed by the hook.
 */
static inline int nf_hook(u_int8_t pf, unsigned int hook, struct net *net,
                          struct sock *sk, struct sk_buff *skb,
                          struct net_device *indev, struct net_device *outdev,
                          int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
    // code omit
}
```
The function `NF_HOOK` contains two steps:
- First, runs the hook's callback functions by calling the underlying function `nf_hook`. 
- Second, invokes the function `okfn` (passed to *NF_HOOK* as the argument), if the packet passes through the hook functions and doesn't drop.

For the hook *NF_INET_LOCAL_IN*, the function `ip_rcv_finish` will be invoked after the hook functions pass. Its job is to pass the packet on to the next protocol handler(TCP or UDP) in the protocol stack to continue its journey! 

The other 4 hooks all use the same function `NF_HOOK` to trigger the callback functions. The following table shows where the hooks are embedded in the kernel, I leave them to the readers. 

| Hook | File | Function |
| ------ | ----------- |----------- |
| NF_INET_PRE_ROUTING | /kernel-src/net/ipv4/ip_input.c      | ip_rcv()                |
| NF_INET_LOCAL_IN    | /kernel-src/net/ipv4/ip_input.c      | ip_local_deliver()      |
| NF_INET_FORWARD     | /kernel-src/net/ipv4/ip_forward.c    | ip_forward()            |
| NF_INET_POST_ROUTING| /kernel-src/net/ipv4/ip_output.c     | ip_build_and_send_pkt() |
| NF_INET_LOCAL_OUT   | /kernel-src/net/ipv4/ip_output.c     |ip_output()              |

Next, Let's review the Netfilter's APIs to create and register the hook function. 
### Netfilter API

It's straightforward to create a Netfilter module, which involves three steps: 
- Define the hook function.
- Register the hook function in the kernel module initialization process.
- Unregister the hook function in the kernel module clean-up process. 

Let's go through them quickly one by one. 

##### Define a hook function

The hook function name can be whatever you want, but it must follow the signature below: 

```c
//In source code file /kernel-src/include/linux/netfilter.h
typedef unsigned int nf_hookfn(void *priv,
                               struct sk_buff *skb,
                               const struct nf_hook_state *state);
```

The hook function can mangle or filter the packet whose data is stored in the `sk_buff` structure (we can ignore the other two parameters; since we don't use them in our mini-firewall). As we mentioned above, the callback function must return a Netfilter status code which is an integer. For instance, the `accepted` and `dropped` status is defined as follows: 


```c
// In source code file /kernel-src/include/uapi/linux/netfilter.h
/* Responses from hook functions. */
#define NF_DROP 0
#define NF_ACCEPT 1
```
##### Register and unregister a hook function
To register a hook function, we should wrap the defined hook function with related information, such as which hook you want to bind to, the protocol family and the priority of the hook function,  into a structure `struct nf_hook_ops` and pass it to the function `nf_register_net_hook`. 

```c
//In source code file /kernel-src/include/linux/netfilter.h
struct nf_hook_ops {
        /* User fills in from here down. */
        nf_hookfn               *hook;    // callback function
        struct net_device       *dev;     // network device interface
        void                    *priv; 
        u_int8_t                pf;       // protocol
        unsigned int            hooknum;  // Netfilter hook enum
        /* Hooks are ordered in ascending priority. */
        int                     priority; // priority of callback function
};
```
Most of the fields are very straightforward to understand. The one need to emphasize is the field `hooknum`, which is just the Netfilter hooks discussed above. They are defined as enumerators as follows: 

```c
// In source code file /kernel-src/include/uapi/linux/netfilter.h
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS,
	NF_INET_INGRESS = NF_INET_NUMHOOKS,
};
```

Next, let's take a look at the functions to register and unregister hook functions goes as follows: 

```c
//In source code file /kernel-src/include/linux/netfilter.h
/* Function to register/unregister hook points. */
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);
```
The first parameter `struct net` is related to the network namespace, we can ignore it for now and use a default value. 

Next, let's implement our mini-firewall based on these APIs. All right? 

### Implement mini-firewall

First, we need to clarify the requirements for our mini-firewall. We'll implement two network traffic control rules in the mini-firewall as follows:
- *Network protocol rule*: drops the [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) protocol packets.
- *IP address rule*: drops the packets from one specific IP address.

The completed code implementation is in this Github [repo](https://github.com/baoqger/linux-mini-firewall-netfilter/blob/main/mini_firewall.c).

##### Drop ICMP protocol packets

`ICMP` is a network protocol widely used in the real world. The popular diagnostic tools like `ping` and `traceroute` run the ICMP protocol. We can filter out the ICMP packets based on the protocol type in the IP headers with the following hook function: 
```c
// In mini-firewall.c 
static unsigned int nf_blockicmppkt_handler(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)
{
	struct iphdr *iph;   // IP header
	struct udphdr *udph; // UDP header
	if(!skb)
		return NF_ACCEPT;
	iph = ip_hdr(skb); // retrieve the IP headers from the packet
	if(iph->protocol == IPPROTO_UDP) { 
		udph = udp_hdr(skb);
		if(ntohs(udph->dest) == 53) {
			return NF_ACCEPT; // accept UDP packet
		}
	}
	else if (iph->protocol == IPPROTO_TCP) {
		return NF_ACCEPT; // accept TCP packet
	}
	else if (iph->protocol == IPPROTO_ICMP) {
		printk(KERN_INFO "Drop ICMP packet \n");
		return NF_DROP;   // drop TCP packet
	}
	return NF_ACCEPT;
}
```

The logic in the above hook function is easy to understand. First, we retrieve the IP headers from the network packet. And then according to the `protocol` type field in the headers, we decided to accept TCP and UDP packets but drop the ICMP packets. The only technique we need to pay attention to is the function `ip_hdr`, which is the kernel function defined as follows: 

```c
//In source code file /kernel-src/include/linux/ip.h
static inline struct iphdr *ip_hdr(const struct sk_buff *skb)
{
        return (struct iphdr *)skb_network_header(skb);
}
// In source code file /kernel-src/include/linux/skbuff.h
static inline unsigned char *skb_network_header(const struct sk_buff *skb)
{
        return skb->head + skb->network_header;
}
```
The function `ip_hdr` delegates the task to the function `skb_network_header`. It gets IP headers based on the following two data: 

- head: is the pointer to the packet;
- network_header: is the offset between the pointer to the packet and the pointer to the network layer protocol header. In detail, you can refer to this [document](https://linux-kernel-labs.github.io/refs/heads/master/labs/networking.html).

Next, we can register the above hook function as follows: 

```c
// In mini-firewall.c 
static struct nf_hook_ops *nf_blockicmppkt_ops = NULL;

static int __init nf_minifirewall_init(void) {
	nf_blockicmppkt_ops = (struct nf_hook_ops*)kcalloc(1,  sizeof(struct nf_hook_ops), GFP_KERNEL);
	if (nf_blockicmppkt_ops != NULL) {
		nf_blockicmppkt_ops->hook = (nf_hookfn*)nf_blockicmppkt_handler;
		nf_blockicmppkt_ops->hooknum = NF_INET_PRE_ROUTING;
		nf_blockicmppkt_ops->pf = NFPROTO_IPV4;
		nf_blockicmppkt_ops->priority = NF_IP_PRI_FIRST; // set the priority
		
		nf_register_net_hook(&init_net, nf_blockicmppkt_ops);
	}
	return 0;
}

static void __exit nf_minifirewall_exit(void) {
	if(nf_blockicmppkt_ops != NULL) {
		nf_unregister_net_hook(&init_net, nf_blockicmppkt_ops);
		kfree(nf_blockicmppkt_ops);
	}
	printk(KERN_INFO "Exit");
}

module_init(nf_minifirewall_init);
module_exit(nf_minifirewall_exit);
```

The above logic is self-explaining. I will not spend too much time here. 

Next, it's time to demo how our mini-firewall works. 

##### Demo time

Before we load the mini-firewall module, the `ping` command can work as expected: 

```python
chrisbao@CN0005DOU18129:~$ lsmod | grep mini_firewall
chrisbao@CN0005DOU18129:~$ ping www.google.com
PING www.google.com (142.250.4.103) 56(84) bytes of data.
64 bytes from sm-in-f103.1e100.net (142.250.4.103): icmp_seq=1 ttl=104 time=71.9 ms
64 bytes from sm-in-f103.1e100.net (142.250.4.103): icmp_seq=2 ttl=104 time=71.8 ms
64 bytes from sm-in-f103.1e100.net (142.250.4.103): icmp_seq=3 ttl=104 time=71.9 ms
64 bytes from sm-in-f103.1e100.net (142.250.4.103): icmp_seq=4 ttl=104 time=71.8 ms
^C
--- www.google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 71.857/71.902/71.961/0.193 ms
```

In contrast, after the mini-firewall module is built and loaded (based on the commands we discussed previously): 

```python
chrisbao@CN0005DOU18129:~$ lsmod | grep mini_firewall
mini_firewall          16384  0
chrisbao@CN0005DOU18129:~$ ping www.google.com
PING www.google.com (142.250.4.105) 56(84) bytes of data.
^C
--- www.google.com ping statistics ---
6 packets transmitted, 0 received, 100% packet loss, time 5097ms
```

You can see all the packets are lost; because it is dropped by our mini-firewall. We can verify this by running the command `dmesg`: 

```python
chrisbao@CN0005DOU18129:~$ dmesg | tail -n 5
[ 1260.184712] Drop ICMP packet
[ 1261.208637] Drop ICMP packet
[ 1262.232669] Drop ICMP packet
[ 1263.256757] Drop ICMP packet
[ 1264.280733] Drop ICMP packet
```

But other protocol packets can still run through the firewall. For instance, the command `wget 142.250.4.103` can return normally as follows:

```python
chrisbao@CN0005DOU18129:~$ wget 142.250.4.103
--2022-06-25 10:12:39--  http://142.250.4.103/
Connecting to 142.250.4.103:80... connected.
HTTP request sent, awaiting response... 302 Moved Temporarily
Location: http://142.250.4.103:6080/php/urlblock.php?args=AAAAfQAAABAjFEC0HSM7xhfO~a53FMMaAAAAEILI_eaKvZQ2xBfgKEgDtwsAAABNAAAATRPNhqoqFgHJ0ggbKLKcdinR4UvnlhgAR4~YyrY4tAnroOFkE_IsHsOg9~RFPc7nEoj6YdiDgqZImAmb_xw9ZuFLvF91P2HzP5tlu1WX&url=http://142.250.4.103%2f [following]
--2022-06-25 10:12:39--  http://142.250.4.103:6080/php/urlblock.php?args=AAAAfQAAABAjFEC0HSM7xhfO~a53FMMaAAAAEILI_eaKvZQ2xBfgKEgDtwsAAABNAAAATRPNhqoqFgHJ0ggbKLKcdinR4UvnlhgAR4~YyrY4tAnroOFkE_IsHsOg9~RFPc7nEoj6YdiDgqZImAmb_xw9ZuFLvF91P2HzP5tlu1WX&url=http://142.250.4.103%2f
Connecting to 142.250.4.103:6080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3248 (3.2K) [text/html]
Saving to: ‘index.html’

index.html                                           100%[===================================================================================================================>]   3.17K  --.-KB/s    in 0s

2022-06-25 10:12:39 (332 MB/s) - ‘index.html’ saved [3248/3248]
```

Next, let's try to ban the traffic from this IP address. 

##### Drop packets source from one specific IP address

As we mentioned above, multiple callback functions are allowed to be registered on the same Netfilter hook. So we will define the second hook function with a different priority. The logic of this hook function goes like this: we can get the source IP address from the IP headers and make the drop or accept decision according to it. The code goes as follows

```c
// In mini-firewall.c 
#define IPADDRESS(addr) \
	((unsigned char *)&addr)[3], \
	((unsigned char *)&addr)[2], \
	((unsigned char *)&addr)[1], \
	((unsigned char *)&addr)[0]

static char *ip_addr_rule = "142.250.4.103";

static unsigned int nf_blockipaddr_handler(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)
{
	if (!skb) {
		return NF_ACCEPT;
	} else {
		char *str = (char *)kmalloc(16, GFP_KERNEL);
		u32 sip;
		struct sk_buff *sb = NULL;
		struct iphdr *iph;

		sb = skb;
		iph = ip_hdr(sb);
		sip = ntohl(iph->saddr); // get source ip address; 
		
		sprintf(str, "%u.%u.%u.%u", IPADDRESS(sip)); // convert to standard IP address format
		if(!strcmp(str, ip_addr_rule)) {
			return NF_DROP;
		} else {
			return NF_ACCEPT;
		}
	}
}
```

This hook function uses two interesting techniques:
- `ntohl`: is a kernel function, which is used to convert the value from `network byte order` to `host byte order`. `Byte order` is related to the computer science concept of [`Endianness`](https://en.wikipedia.org/wiki/Endianness). Endianness defines the order or sequence of bytes of a word of digital data in computer memory. A `big-endian` system stores the most significant byte of a word at the smallest memory address.  A `little-endian` system, in contrast, stores the least-significant byte at the smallest address. Network protocol uses the `big-endian` system. But different OS and platforms run various Endianness system. So it may need such conversion based on the host machine.

- `IPADDRESS`: is a macro, which generates the standard IP address format(four 8-bit fields separated by periods) from a 32-bit integer. It uses the technique of [`the equivalence of arrays and pointers in C`](https://www.eskimo.com/~scs/cclass/notes/sx10e.html). I will write another article to examine what it is and how it works. Please keep watching my updates!

Next, we can register this hook function in the same way discussed above. The only remarkable point is this callback function should have a different priority as follows:

```c
static int __init nf_minifirewall_init(void) {
	<-omit code->
	nf_blockipaddr_ops = (struct nf_hook_ops*)kcalloc(1, sizeof(struct nf_hook_ops), GFP_KERNEL);
	if (nf_blockipaddr_ops != NULL) {
		nf_blockipaddr_ops->hook = (nf_hookfn*)nf_blockipaddr_handler;
		nf_blockipaddr_ops->hooknum = NF_INET_PRE_ROUTING;  // register to the same hook
		nf_blockipaddr_ops->pf = NFPROTO_IPV4;
		nf_blockipaddr_ops->priority = NF_IP_PRI_FIRST + 1; // set a higher priority

		nf_register_net_hook(&init_net, nf_blockipaddr_ops);
	}
	<-omit code->
}
```

Let's see how it works with a demo. 
##### Demo time

After re-build and re-load the module, we can get: 

```python
chrisbao@CN0005DOU18129:~$ wget 142.250.4.103
--2022-06-25 10:20:07--  http://142.250.4.103/
Connecting to 142.250.4.103:80... failed: Connection timed out.
Retrying.
```

The `wget 142.250.4.103` can't return response. Because it is dropped by our mini-firewall. Great!

```python
chrisbao@CN0005DOU18129:~$ dmesg | tail -n 5
[ 3162.064284] Drop packet from 142.250.4.103
[ 3166.089466] Drop packet from 142.250.4.103
[ 3166.288603] Drop packet from 142.250.4.103
[ 3174.345463] Drop packet from 142.250.4.103
[ 3174.480123] Drop packet from 142.250.4.103
```

### More space to expand

You can find the full code implementation [here](https://github.com/baoqger/linux-mini-firewall-netfilter/blob/main/mini_firewall.c). But I have to say, our mini-firewall only touches the surface of what Netfilter can provide. You can keep expanding the functionalities. For example, currently, the rules are hardcoded, why not make it possible to config the rules dynamically. There are many cool ideas worth trying. I leave it for the readers.  

### Summary
In this article, we implement the mini-firewall step by step and examined many detailed techniques. Not only code; but we also verify the behavior of the mini-firewall by running real demos.   

