---
title: Why TCP connection termination needs four-way handshake
date: 2019-7-14 16:34:58
tags: TCP, IP
---

### Background

In `TCP/IP` protocol stack, the `three way handshake` for connection establishment and `four way handshake` for connection termination are some concepts you must understand.

But after I learned `TCP/IP` stack, one question coming to my mind is why the connection termination needs four-way handshake. In this article, I will focus on this topic. To understand the answer of this question, we need to first explain the details of these two processes and compare them to find the difference. 

### TCP connection establishment

In contrast to `UDP`, TCP is a `connection-oriented` and `reliable full duplex` protocol, which requires a logical connection to be established between the two processes before data is exchanged. The connection must be maintained during the entire process that communication is taking place. And both sides can send data to its peer. 

Four things need to happen for fully establishing a TCP connection:
- Each node has send a `SYN` flag to its peer (that's two things, one `SYN` in each direction)
- Each node has received a `ACK` flag of its own `SYN` flag from its peer  (that's two things) 

Both `SYN` and `ACK` are properties in `TCP` header, where `SYN` means `synchronization` and `ACK` means `acknowledgement`. In detail the process goes as follows:

Firstly, the client sends the `SYN: 1` packet to server to start the connection. When the server receives the packet, it will send the `ACK: 1` packet to client to confirm that. Since in the TCP protocol, the server can also send data to the client. So the server needs to send the `SYN: 1` packet to the client as well. Finally, when the client sends the `ACK: 1` packet to the server. The establishment process is done. And the process goes as follows: 

<img src="/images/tcp-establishment-4-packets.png" title="tcp establishment with four packets" width="600px" height="400px">

Note that the server sends two packets: `ACK: 1` and `SYN: 1` in sequence. Why not combine these two packets into one to have a better network performance? So the common implementation of TCP protocol  goes as follows:

<img src="/images/tcp-establishment-3-packets.png" title="tcp establishment with three packets" width="600px" height="400px">

<img src="/images/tcp-termination.png" title="tcp termination" width="800px" height="600px">

**Note**: besides `SYN` and `ACK` flag, in the establishment stage there are other properties need to be set in TCP header such as: `sequence number` and `window`. I will discuss these important TCP header properties in other articles.   

### TCP connection termination

outline:

the workflow of 3 way handshake
    4 packets model
    3 packets model

the workflow 
    4 packets model
    TCP status change

overall summary: need to exchange 4 information
