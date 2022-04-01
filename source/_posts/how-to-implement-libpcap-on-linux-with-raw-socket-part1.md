---
title: "Write a Linux packet sniffer from scratch: part one- PF_PACKET socket and promiscuous mode"
date: 2022-02-22 10:21:14
tags: packet sniffer, socket, PF_PACKET
---

### Background

When we refer to network packet sniffer, some famous and popular tools come to your mind, like `tcpdump`. I have shown you how to capture network packets in my previous articles. But have you ever thought about writing a packet sniffer from scratch? In this article, let us do it. After reading this article, you can find that it is not as difficult as you think. 

### Introduction

Firstly, we need to review how `tcpdump` is implemented. According to the official [document](https://www.tcpdump.org/), `tcpdump` is built on the library `libpcap`, which is developed based on the remarkable research result from Berkeley, in details you can refer to [this paper](https://www.tcpdump.org/papers/bpf-usenix93.pdf).

As you know, different operating systems have different internal implementations of network stacks. `libpcap` covers all of these differences and provides the system-independent interface for user-level packet capture. I want to focus on the Linux platform, so how does `libpcap` work on the Linux system? According to some [documents](https://stackoverflow.com/questions/21200009/capturing-performance-with-pcap-vs-raw-socket), it turns out that libpcap uses the `PF_PACKET` socket to capture packets on a network interface.

So the next question is: what the `PF_PACKET` socket is? 

### PF_PACKET socket

In my previous [article](https://organicprogrammer.com/2021/07/31/how-to-implement-simple-http-server-golang/), we mentioned that the socket interface is TCP/IP’s window on the world. In most modern systems incorporating TCP/IP, the socket interface is the only way applications can use the TCP/IP suite of protocols. 

<img src="/images/pf-packet-socket.png" title="PF_PACKET socket" width="400px" height="300px">

It is correct. This time, let's dig deeper about `socket` by examining the system call executed when we create a new socket: 

```c
int socket(int domain, int type, int protocol);
```

When you want to create a socket with the above system call, you have to specify which domain (or protocol family) you want to use with that socket as the first argument. The most commonly used family is `PF_INET`, which is for communications based on IPv4 protocols (when you create a TCP server, you use this family). Moreover, you have to specify a type for your socket as the second argument. And the possible values depend on the family you specified. For example, when dealing with the `PF_INET` family, the values for type include `SOCK_STREAM`(for TCP) and `SOCK_DGRAM`(for UDP). For other detailed information about the socket system call, you can refer to the socket(3) man page. 

You can find one potential value for the `domain` argument as follows:

```
 AF_PACKET    Low-level packet interface
```
**Note**: `AF_PACKET` and `PF_PACKET` are same. It is called `PF_PACKET` in history and then renamed  `AF_PACKET` later. `PF` means protocol families, and `AF` means address families. In this article, I use `PF_PACKET`. 

Different from `PF_INET` socket, which can give you TCP segment. By `PF_PACKET` socket, you can get the raw `Ethernet` frame which bypasses the usual upper layer handling of TCP/IP stack. It might sound a little bit crazy. But, that is, any packet received will be directly passed to the application. 

For a better understanding of `PF_PACKET` socket, let us go deeper and roughly examine the path of a received packet from the network interface to the application level. 

(As shown in the image above) When the network interface card(NIC) receives a packet, it is handled by the driver. The driver maintains a structure called `ring buffer` internally. And write the packet to kernel memory (the memory is pre-allocated with ring buffer)  with direct memory access(DMA). The packet is placed inside a structure called **`sk_buff`**(one of the most important structures related to kernel network subsystem).   

After entering the kernel space, the packet goes through protocol stack handling layer by layer, such as `IP processing` and `TCP/UDP processing`. And the packet goes into applications via the socket interface. You already understand this familiar path very well.

But for the `PF_PACKET` socket, the packet in `sk_buff` is cloned, then it skips the protocol stacks and directly goes to the application. The kernel needs the clone operation, because one copy is consumed by the `PF_PACKET` socket, and the other one goes through the usual protocol stacks.

In future articles, I'll demonstrate more about Linux kernel network internals.

Next step, let us see how to create a `PF_PACKET` socket at the code level. For brevity, I omit some code and only show the essential part. You can refer to this Github [repo](https://github.com/baoqger/raw-socket-packet-capture-/blob/master/raw_socket.c) in detail.  

```cpp
    if ((sock = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP))) < 0) {
        perror("socket");
        exit(1);
    }
```

Please ensure to include the system header files: `<sys/socket.h> <sys/types.h>`. 

### Bind to one network interface

Without the additional settings, the sniffer captures all the packets received on all the network devices. Next step, let us try to bind the sniffer to a specific network device. 

Firstly, you can use `ifconfig` command to list all the available `network interfaces` on your machines. The network interface is a software interface to the networking hardware. 

For example, the following image shows information of network interface `eth0`: 

```s
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.230.49  netmask 255.255.240.0  broadcast 192.168.239.255
        inet6 fe80::215:5dff:fefb:e31f  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:fb:e3:1f  txqueuelen 1000  (Ethernet)
        RX packets 260  bytes 87732 (87.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 178  bytes 29393 (29.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Let's bind the sniffer to `eth0` as follows:

```c
    // bind to eth0 interface only
    const char *opt;
    opt = "eth0";
    if (setsockopt(sock, SOL_SOCKET, SO_BINDTODEVICE, opt, strlen(opt) + 1) < 0) {
        perror("setsockopt bind device");
        close(sock);
        exit(1);
    }
```
We do it by calling the `setsockopt` system call. I leave the detailed usage of it to you. 

Now the sniffer only captures network packets received on the specified network card. 

### Non-promiscuous and promiscuous mode

By default, each network card minds its own business and reads only the frames directed to it. It means that the network card discards all the packets that do not contain its own MAC address, which is called `non-promiscuous` mode. 

Next, let us make the sniffer can work in `promiscuous` mode. In this way, it retrieves all the data packets. Even the ones that are not addressed to its host. 

To set a network interface to promiscuous mode, all we have to do is issue the `ioctl()` system call to an open socket on that interface.

```c
    /* set the network card in promiscuos mode*/
    // An ioctl() request has encoded in it whether the argument is an in parameter or out parameter
    // SIOCGIFFLAGS	0x8913		/* get flags			*/
    // SIOCSIFFLAGS	0x8914		/* set flags			*/
    struct ifreq ethreq;
    strncpy(ethreq.ifr_name, "eth0", IF_NAMESIZE);
    if (ioctl(sock, SIOCGIFFLAGS, &ethreq) == -1) {
        perror("ioctl");
        close(sock);
        exit(1);
    }
    ethreq.ifr_flags |= IFF_PROMISC;
    if (ioctl(sock, SIOCSIFFLAGS, &ethreq) == -1) {
        perror("ioctl");
        close(sock);
        exit(1);
    }
```

`ioctl` stands for **I/O control**, which manipulates the underlying device parameters of specific files. `ioctl` takes three arguments: 
- The first argument must be an open file descriptor. We use the socket file descriptor bound to the network interface in our case.
- The second argument is a device-dependent request code. You can see we called `ioctl` twice. The first call uses request code *SIOC**G**IFFLAGS* to get flags, and the second call uses request code *SIOC**S**IFFLAGS* to set flags. Do not be fooled by these two constant values, which are spelled alike.
- The third argument is for returning information to the requesting process.  

Now the sniffer can retrieve all the data packets received on the network card, no matter to which host the packets are addressed.
### Summary

This article examined what `PF_PACKET` socket is, how it works and why the application can get raw Ethernet packets. Furthermore, we discussed how to bind the sniffer to one specific network interface and how can make the sniffer work in the promiscuous mode. The next article will examine how to implement the packet filter functionality, which is very useful to a network sniffer. 
