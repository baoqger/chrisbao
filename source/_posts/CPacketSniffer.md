---
title: cPacketSniffer
date: 2022-07-22 10:19:48
tags: network packet capture, libpcap, analyze and track network traffics, detect network security attacks, Linux system programming
keywords: network packet capture, libpcap, analyze and track network traffics, detect network security attacks, Linux system programming
---

### Background
In this post, I want to introduce my new project: [`cPacketSniffer`](https://github.com/baoqger/cPacketSniffer). I worked on it for the past two months. Finally, I worked it out and feel very proud of putting it here!

### Motivation
Simply speaking, I want to sharpen my techniques in `network programming` and `Linux system programming`. Both of these two topics can lead you to the bottom of computers or software. Feynman said ["There is plenty of room at the bottom"](https://en.wikipedia.org/wiki/There%27s_Plenty_of_Room_at_the_Bottom), I think this physics law can apply to software as well. 

### Acknowledgement 

It's very lucky for me to come across this site ["Network programming in Linux"](http://tcpip.marcolavoie.ca/index.html), which developed a network packet capturing tool with `C++`. After confirming that the documents and source code on this site is completed and clear, I decided to refactor it with C language. That's the starting point for my project `cPacketSniffer`.      

### Features

As a network packets sniffer, `cPacketSniffer` provides the following features: 

<img src="/images/cPacketSniffer.png" title="cPacketSniffer modules" width="600px" height="400px">

- Integrate with `libpcap` to support: filtering captured packets, capturing packets offline, capturing packets on specific devices and capturing packets in promiscuous mode.
- Analyze network packets at low layers of TCP/IP stack, including `Ethernet`, `ARP`, `ICMP`, `IP(IPv4)`, `TCP`, `UDP`, etc. Also one protocol in the application layer: `TFTP`. 
- Detect network security attacks:
    - ARP spoofing detection.
    - Ping flood detection.
- Analyze and track network traffics:
    - TCP session tracking and traffic analysis.
    - TFTP session tracking and traffic analysis.

The following images demonstrate some typical usages of `cPacketSniffer`:  

**Packet Analysis**:

<img src="/images/packet-analysis.png" title="packet analysis" width="600px" height="400px">

**ARP Spoofing Detection**:

<img src="/images/arpspoof.png" title="arp spoofing detection" width="600px" height="400px">

**PING Flood Detection**:

<img src="/images/pingflooddetection.png" title="ping flood detection" width="600px" height="400px">

**TCP Session Tracking**:

<img src="/images/tcpsessiontrack.png" title="tcp session track" width="600px" height="400px">

Besides the above network programming-related functionalities, it also covers the following points: 
- Develop a generic data structure in C.
- Error handling in C. 
- Data encapsulation (object-oriented style programming) in C.
- Manual memory management in C.
- etc.

This article will not cover these points in detail, I will write articles on these topics separately in the future. Please keep watching my blog!

### Future work

Now `cPacketSniffer` can work as a network packet sniffer based on the design. Moreover, it can also serve as a testbed to try experimental features. Next step I plan to try the following ideas:

- Implement the network intrusion detection function. 
- Improve the performance with advanced data structures, like binary search trees. 
- Memory and cache performance tuning. 
- Automatic memory management by Garbage Collection.
- Integrate `ncurses` for Text-based user interface.






