---
title: "How HTTPS works: part four - digital signature"
date: 2019-3-5 16:00:47
tags: digital signature, hash function
---
### Background

The [second](https://organicprogrammer.com/2019/02/25/https-certificate/) article of this series shows that we should not directly exchange `public key` to establish a secure channel. Instead, `certificates` should be used to establish trust between clients and servers. And the last article examines the content of certificates, which encodes two very crucial pieces of information: **the server’s public key** and a **digital signature** that can confirm the certificate’s authenticity. In this article, let us examine what digital signature is and how it works.  

### Digital Signature

The concept of `digital signature` can be illustrated as follows: 

<img src="/images/https-signature-process.png" title="digital signature process" width="600px" height="400px">

When the server sends a message to the client, how can we prevent the attackers from eavesdropping on the message? The server can attach its signature to it and the client can accept the message only if the verification of the certificate passed. So this process can be devided into two parts: `sign the message to get the digital signature` and `verify the digital signature`.

- **Sign the message**: means `hash` the message and `encrypt` the hash with the server's private key. The encrypted hash is called `digital signature`. 

- **Verify the signature**: means `decrypt` the signature with the server's public key to get the hash, `re-compute` another hash of the message and compare identity of two hashes.

What can we benefit from signing and verifing the digital signature? To verify the digital signature is to confirm two things: 

- **Integrity**: the message has not changed since the signature was attached because it is based on a  `cryptographic hash` of the message. 

- **Authenticity**: the signature belongs to the person who alone has access to the private key. 

Information security has other attributes, but integrity and authentication are the two traits you must know.

**Hash is irreversible**

Unlike cryptographic algorithms, though, message digests do not have to be reversible – in fact, this irreversibility is the whole point.

The goal of MD5 or any secure hashing algorithm is to reduce the arbitrarily sized input into an n-bit hash in such way that it is very unlikely that two messages, regardless of length or content, produce identical hashes – that is, collide – and that is impossible to specifically reverse engineer such a collision.

For md5, n = 128 bits. This means that there are 2^128 possible MD5 hashes. Although the input space is vastly larger than this, 2^128 makes it highly unlikely that two messages will share the same MD5 hash. More importantly, it should be impossible, assuming that MD5 hashes are evenly, randomly distributed, for an attacker to compute a useful message that collides with another by way of brute force. 

**the private key can also be used to prove identity**
At first glance, this doesn’t sound very useful. The public key, after all, is
public. It’s freely shared with anybody and everybody. Therefore, if a value is
encrypted with the private key, it can be decrypted by anybody and everybody as
well. However, the nature of public/private keypairs is such that it’s also impossible — or, to be technically precise, mathematically infeasible — for anybody
except the holder of the private key to generate something that can be decrypted
using the public key. After all, the encryptor must find a number c such that
ce
%n  m for some arbitrary m. By definition, c  md satisfies this condition and
it is believed to be computationally infeasible to find another such number c.
As a result, the private key can also be used to prove identity. 

思路：书接上文，digital signature。

首先要介绍digital signuture的sign和verify技术实现。通过配图解释。

通过对实现过程的具体解释，就能理解它能实现的secure效果: integrity和authenticity. 
To verify the digital signature is to confirm two things: first, that the vouch-for artifact has not changed since the signature was attached because it is based, in part, on a cryptographic hash of the document. Second, that the signature belongs to the person who alone has access to the private key in a pair. 

技术点解释：其间要大致讲清的关键点：什么是sign the message? hash和encryption的区别?(hash是不可逆的)。public-key本身就有验证身份的能力（数学原理带来的）。digital signature的广泛应用举例子。

上面是general的digital signature情况。需要放到x.509 certificate的场景中具体讨论下，它是如何让certificate安全的
再通过两个实例理解：第一attackers forge certificate的内容；第二attackers replace with its own certificate

另外，signature是基于certificate的整体信息得到的，不只是public key, 这一点也要说明。否则不能避免上面的第二种攻击。因为之前的文章说了public key不能直接exchange，需要certificate来包裹一层。所以最开始我以为signature就是基于public key算出来的


第二篇介绍 digital signature

第三篇介绍 手动验证signature

第四篇介绍 trusts/certificates chain


extension:
certificate除了能够防止man-in-the-middle attack, 还有其他3个功能(包括revoke等). 


