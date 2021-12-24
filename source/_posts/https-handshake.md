---
title: "How HTTPS works: part one - TLS handshake"
date: 2019-2-22 11:04:58
tags: HTTPS, TLS handshake, SSL/TLS
---

### Background

Nowdays everybody knows about the lock icon in the browser's address bar indicating that the session is protected by `HTTPS`. In this series articles, I'll show you how HTTPS works based on my research. 

Simply speaking, `HTTPS = HTTP + SSL/TLS`. 

As we know, in `HTTP` protocol the client and server communicate with each other by transmitting the message over the network. But the message is in the original format which is known as plaintext. The bad guys (attacker) can easily eavesdrop on the message. That's the reason why we need `SSL/TLS`. 

There are tons of [articles](https://www.globalsign.com/en/blog/ssl-vs-tls-difference) online describing the relationship between SSL and TLS. You can regard TLS is the morden and advanced version of SSL. By conversion, we generally put them together as the name of the network security protocol. 

In the [OSI model](https://en.wikipedia.org/wiki/OSI_model), `HTTP` protocol is in **layer 7**, which is at the top of protocol stacks, how about `TLS` protocol. Although `TLS` is short for `Transport Layer Security`, it's not in **layer 4(transport layer)**. You can roughly regard it as **layer 5(session layer)**. 

This means there is no nontrivial difference between `HTTP` and `HTTPS` in terms of message transmission. This sereis of article will focus on the security part of `HTTPS`. If you want to understand about HTTP itself, please read my other [articles](https://baoqger.github.io/2021/12/01/understand-http-1-1-client-golang/). 

As the first article of this series, I will talk about how to establish an `HTTPS` connection. This introduce a new concept: **TLS handshake**.

### TLS Handshake

In my previous article, I wrote about [`TCP three way handshake`](https://baoqger.github.io/2019/07/14/why-tcp-four-way-handshake/) to establish a TCP connection. In `HTTP` protocol, that's all it needs to do. But for `HTTPS`, after the `TCP handshake` it has to do `TLS handshake` as well. 

<img src="/images/https-tls-handshake.png" title="tls handshake" width="600px" height="400px">

**Note** to understand each detail of the above processing, you need have some prerequisite knowledges such as `encryption/decryption`, `hash function`, `public key`, `private key`, `digital certificate` and so on.  

If you don't know, don't worry. After reading this series of article, you can say you already had it under your belt. 

TLS handshake （只能先general 介绍，细节的坑太多）

从上面引出一些问题：

1. tls同时用了对称和非对称加密(handshake用了非对称，session的过程用了对称加密)
2. 为什么对message加密要采用对称加密算法？是因为性能吗？
3. 不能直接传递public key => 所以下一个问题就是certificate的理解
4. 解析certificate的内容结构
5. 什么是digital signature? 什么是sign the hash
6. certifcate能解决secure的哪些问题？
7. 下面就是certificate chain的问题？为什么需要trust chain?
8. 手动(用openssl)verify一个certificate
