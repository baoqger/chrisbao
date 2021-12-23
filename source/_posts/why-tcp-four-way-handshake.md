---
title: Why TCP connection termination needs four-way handshake
date: 2019-1-14 16:34:58
tags: TCP, IP
---

### Background

In `TCP/IP` protocol stack, the `three way handshake` for connection establishment and `four way handshake` for connection termination are some concepts you must understand.

But after I learned `TCP/IP` stack, one question coming to my mind is why the connection termination needs four-way handshake. In this article, I will focus on this topic. To understand the answer to this question, we need to first explain the details of these two processes and compare them to find the difference. 

### TCP connection establishment

In contrast to `UDP`, TCP is a `connection-oriented` and `reliable full duplex` protocol, which requires a logical connection to be established between the two processes before data is exchanged. The connection must be maintained during the entire process that communication is taking place. And in TCP both sides can send data to its peer. 

So four things need to happen for fully establishing a TCP connection:
- Each node has send a `SYN` flag to its peer (that's two things, one `SYN` in each direction)
- Each node has received a `ACK` flag of its own `SYN` flag from its peer  (that's two things) 

Both `SYN` and `ACK` are properties in `TCP` header, where `SYN` means `synchronization` and `ACK` means `acknowledgement`. In detail the process goes as follows:

Firstly, the client sends the `SYN: 1` packet to server to start the connection. When the server receives the packet, it will send the `ACK: 1` packet to client to confirm that. Since in the TCP protocol, the server can also send data to the client. So the server needs to send the `SYN: 1` packet to the client as well. Finally, when the client sends the `ACK: 1` packet to the server. The establishment process is done, which goes as follows: 

<img src="/images/tcp-establishment-4-packets.png" title="tcp establishment with four packets" width="600px" height="400px">

Note that the server sends two packets: `ACK: 1` and `SYN: 1` in sequence. Why not combine these two packets into one to have a better network performance? So the common implementation of TCP protocol  goes as follows:

<img src="/images/tcp-establishment-3-packets.png" title="tcp establishment with three packets" width="600px" height="400px">

**Note**: besides `SYN` and `ACK` flag, in the establishment stage there are other fields need to be set in TCP header such as: `sequence number` and `window`. I will discuss these important TCP header fields in other articles.

### TCP connection termination

When the data stream transportation (I will not cover this part in this article) is over, we need to terminate the TCP connection. 

Same as above, four things need to happen for fully terminating a TCP connection:
- Each node has send a `FIN` flag to its peer (that's two things, one `FIN` in each direction)
- Each node has received a `ACK` flag of its own `FIN` flag from its peer  (that's two things) 

<img src="/images/tcp_termination.png" title="tcp termination" width="600px" height="400px">

But the difference is that in most TCP implementation, four packets are used to terminate the connection. We didn't combine the `ACK: 1` packet (marked as ②) and the `FIN: 1` packet (marked as ③) into one packet. 

The reason is `ACK: 1` packet (marked as ②) is send by TCP stack automatically. And the next `FIN: 1` packet (marked as ③) is controlled in application level by calling `close` socket API. Application has the control to terminate the connection. So in common case, we didn't merge this two packets into one. 

It's flexible for the application to control this process, for example, in this way the application can reuse existing TCP connection to reduce the overhead and improve the performance. In next article, I will talk about this topic.  
