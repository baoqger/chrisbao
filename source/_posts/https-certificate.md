---
title: "How HTTPS works: part two - why we need public key and certificate"
date: 2019-2-25 16:00:47
tags: public key, man-in-the-middle, certificate
---

### Background

In the first article of this series, I gave a high-level overview of `TLS handshake`. It covers many complex concepts like `encryption/decryption` `hash function` `public key` `private key`  and `digital certificate`. These concepts are defined in the framework of `Public Key Infrastructure (PKI)`.

Let' continue exploring the PKI. This article focuses on understanding the `certificates` used to establish trust between clients and servers.  

### Symmetric plus Asymmetric

[Last post](https://organicprogrammer.com/2019/02/22/https-handshake/) examined the `TLS handshake`. What is remarkable in this process is both `symmetric encryption` and `asymmetric enecrption` are used. 

`symmetric encryption` and `asymmetric encryption` are the two broad categories of cryptographic algorithms. 

Symmetric key encryption uses the same key on both sides of the communication channel to encrypt or decrypt data. In `HTTPS` case, the client and server agree upon new keys to use for symmetric encryption, called **session key**. The HTTP messages are encrypted by this symmetric session key. 

It has the advantage of being very fast with low overhead. In this way, it can minimize the performance impact `TLS` will have on the network communications. 

But the real challenge of symmetric encryption is **how to keep the key private and secure!** The client and server must exchange keys without letting an interested eavesdropper see them. This seems like a **chicken and egg problem**; you can't establish keys over an insecure channel, and you can't establish a secure channel without keys. Right? 

This key management turns out to be the most difficult part of encryption operations and is where asymmetric or public-key cryptography enters. 

Simply speaking, **a secure channel is firstly established with asymmetric cryptography and then the symmetric session key is exchanged through this secure channel**. 

**Note**: Cryptography is a complex topic, which isn't in the scope of this series of articles. You can find [tons](https://opensource.com/article/19/6/cryptography-basics-openssl-part-2) of documents on the internet about it for deeper understanding. You can regard it as a block box and ignore the details. This doesn't influence your understanding of `HTTPS` in high level 

### Why do we need certificates

Now we know public-key cryptography is needed to establish a secure channel. Can we directly transmit the public key from servers to clients? Why do we need a certificate as the carrier to pass the public key? Can we exchange the public key without certificates as follows: 

<img src="/images/https_wo_certificate.png" title="public key without certificate" width="600px" height="400px">

But the ultimate question is how you(as the client) know that the public key can be trusted as authentic? Assume that an attacker can not only view traffic, but also can intercept and modify it. Then the attacker can carry out `man-in-the-middle` attack as follows:  

<img src="/images/https_mitm.png" title="man in the middle attack" width="800px" height="600px">

The attacker can replace the server's public key with its own and send it to the client. The client doesn't feel anything wrong and keeps using this public key as normal. The client encrypts the session key with the forged public key(the one from attackers) and sends it out. The attacker decrypts the session key with its private key, re-encrypt the session key with the server's public key, and sends it to the server. As normal, the server decrypts the session key and agrees on it. But this session key is in the attacker's hand too. The attacker can decrypt all the subsequent network traffic. 

The problem here is that the client blindly **trusts** that the public key belongs to the server. That's the reason why we need the `certificates` to establish trust between clients and servers.

### Summary
In this article, we understand the importance of public-key cryptography and certificates. In the next article, we will take a deep look at the certificate.  


