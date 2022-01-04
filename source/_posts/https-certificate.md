---
title: https-certificate
date: 2019-2-25 16:00:47
tags:
---

### Background

In the first article of this series, I gave a high-level overview of `TLS handshake`. It covers many complex concepts like `encryption/decryption` `hash function` `public key` `private key`  and `digital certificate`. These concepts are defined in the framework of `Public Key Infrastructure (PKI)`.

Let' continue exploring the PKI. This article focuses on understanding the `certificates` used to establish trust between clients and servers.  

### Symmetric plus Asymmetric

[Last post](https://organicprogrammer.com/2019/02/22/https-handshake/) examined the `TLS handshake`. What is remarkable in this process is both `symmetric encryption` and `asymmetric enecrption` are used. 

`symmetric encryption` and `asymmetric enecrption` are the two broad categories of cryptographic algorithms. 

Symmetric key encryption uses the same key on both sides of the communication channel to encrypt or decrpt data. In `HTTPS` case, the client and server agree upon new keys to use for symmetric encryption, called **session key**. The HTTP messages are encrypted by this symmetric session key. 

It has the advantage of being very fast with a low overhead. In this way it can minimize the performance impact `TLS` will have on the network communications. 

But the real challenge of symmetric encryption is **how to keep the key private and secure!** The client and server must exchange keys without letting an interested eavesdropper see them. This seems like a **chicken and egg problem**; you can't establish keys over an insecure channel, and you can't establish a secure channel without keys. Right? 

This key management turns out to be the most difficult part of encryption operations and is where asymmetric or  public-key cryptography enters. 

Simply speaking, **a secure channel is firstly established with asymmetric cryptography and then the symmetric session key is exchanged through this secure channel**. 

**Note**: Cryptography is a complex topic which is not in the scope of this series of articles. You can find tons of documents on the internet about it for deeper understanding. You can regard it as a block box and ignore the details. This doesn't influence your understanding about `HTTPS` in high level 

### Why we need certificate

Now we know public-key cryptography is needed to establish secure channel. Can we directly transmit the public key from servers to clients? Why we need a certificate as the carrier to pass the public key? 

The ultimate question is how you(as the client) know that the public key can be trusted as authentic?



应该是这个思路：
1 tls握手的过程证明，同时需要对称和非对称加密。这个握手过程本质是通过证书传递public key. 
对称加密, 需要非对称加密来提供channel


2 灵魂发问 can we directly transmit the public key from servers to clients? 
But, how do you(as the client) know that the public key can be trusted as authentic?
引出certifcate

3 certificate是secure的，解决了几个方面的问题，点到为止，具体看文献 https://opensource.com/article/19/6/cryptography-basics-openssl-part-1。

4 certificate的内容
PEM format
用openssl convert to plain text
大致分析它的内容

５　数值签名（值得单独一篇文章）

ｃｈａｉｎ图
搞几个ｃｅｒｔｉｆｉｃａｔｅ放到ｇｉｓｔ上
解释ｃｈａｉｎ的逻辑

压住下面几个关键点会显得更专业：
PKI,
X.509
Trusted chain
PEM format
digital signature

一些不易察觉的知识点：
certificate chain 和root certificate分别在哪里？
为什么需要chain

金句：
A certificate is a standard way to wrap the server's public key, along with its identity and a signature by a trusted authority.

Understanding the X.509 certificate, which is fully defined in RFC 5280, is key to making sense of those errors. Unfortunately, these certificates have a well deserved reputation of being opaque and difficult to manage. With the multitude of formats used to encode them, this reputation is rightly deserved.

The certificate encodes two very important pieces of information: the server's public key and a digital signature that can be used to confirm the certificate's authenticity.  







A hash, often using the SHA256 algorithm, is a digital fingerprint(🤔有数字指纹的说法吗？) of the data. If you change a single bit in the data, the hash will change. By computing a hash over the DER-encoded public key section of the certificate and then signing the hash with its own private key(思考🤔sign the hash? 就是加密的意思吧), the CA is giving its stamp of approval on the certifcate. This signed hash value is the signature appended to the certificate. 

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
