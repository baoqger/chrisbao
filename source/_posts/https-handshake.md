---
title: "How HTTPS works: part one - TLS handshake"
date: 2019-2-22 11:04:58
tags: HTTPS, TLS handshake, SSL/TLS
---

### Background

At present, everybody knows about the lock icon in the browser's address bar indicating that the session is protected by `HTTPS`. In this series of articles, I'll show you how HTTPS works based on my research. 

Simply speaking, `HTTPS = HTTP + SSL/TLS`. 

As we know in the `HTTP` protocol, the client communicates with the server by transmitting the message over the network. But the message is in the original format known as plaintext. The bad guys (attackers) can easily eavesdrop on the message. That's the reason why we need `SSL/TLS`. 

There are tons of [articles](https://www.globalsign.com/en/blog/ssl-vs-tls-difference) online describing the relationship between SSL and TLS. You can regard TLS as the modern and advanced version of SSL. By conversion, we generally put them together as the name of the network security protocol. 

In the [OSI model](https://en.wikipedia.org/wiki/OSI_model), the `HTTP` protocol is in **layer 7**, which is at the top of protocol stacks, how about `TLS` protocol. Although `TLS` is short for `Transport Layer Security`, it's not in **layer 4(transport layer)**. You can roughly regard it as **layer 5(session layer)**. 

This means there is no nontrivial difference between `HTTP` and `HTTPS` in terms of message transmission. This series of articles will focus on the security part of `HTTPS`. If you want to understand HTTP itself, please read my other [articles](https://baoqger.github.io/2021/12/01/understand-http-1-1-client-golang/). 

As the first article of this series, I will talk about how to establish an `HTTPS` connection. This introduces a new concept: **TLS handshake**.

### TLS Handshake

In my previous article, I wrote about [`TCP three way handshake`](https://baoqger.github.io/2019/07/14/why-tcp-four-way-handshake/) to establish a TCP connection. For `HTTP` protocol, that's all it needs to do. But `HTTPS` has to do `TLS handshake` as well. The process can be illustrated as follows: 

**Note** TLS handshakes occur after a TCP connection has been opened via a TCP handshake.

<img src="/images/https-tls-handshake.png" title="tls handshake" width="600px" height="400px">

**Note** to understand each detail of the above processing, you need to have some prerequisite knowledge such as `encryption/decryption`, `hash function`, `public key`, `private key`, and `digital certificate`.  If you don't know, don't worry. After reading this series of articles, you'll have it under your belt. In this article, I'll go through this process roughly and ignore the detail of each step. 

- **Client hello message**: the client starts the handshake by sending a "hello" message to the server. This "hello" message includes the TLS version and the `cipher suites` supported on the client-side. `cipher suites` is a fancy name for the set of encryption algorithms for use in establishing a secure connection.

- **Server hello message**: the server chooses the best(or suitable) SSL/TLS version and encryption algorithm among the candidates sent by the client. The server's chosen cipher suite and `certificate` are sent to the client. `certificate` makes SSL/TLS encryption possible, which is critical to understand the entire process. For now, you need to know certificates contain the server's `public key`  which is sent to the client. And the `private key` is kept secret on the server. 

- **Authentication**: The client verifies the server's certificate (I'll investigate the certificate verification process at a very detailed level in another article). 

- **Encrypt the pre-master key**: The client generates one random "pre-master" key, encrypts "pre-master" key with the server's public key (extract from the certificate) and sends the encrypted "pre-master" key to the server.

- **Decrypt the pre-master key**: The server decrypts the "pre-master" key with its private key (The premaster key is encrypted with the public key and can only be decrypted with the private key by the server. This is called `asymmetric encryption`).

- **Session key created**: Both client and server will generate a `session key` based on the "pre-master" key. The `session key` should be the same on both sides.

- **HTTP message encryption**: Until the last step, the handshake is completed. The client and server will use the agreed `session key` to encrypt the following HTTP messages and continue the communication (since both sides use the same key, this is called `symmetric encryption`). 

That's all about `TLS handshake` process. 

### Summary

As the first article of this series, I go through the entire process of TLS handshake. In future articles, I'll show you the mysteries of each part. 




