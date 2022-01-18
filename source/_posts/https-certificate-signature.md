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

When the server sends a message to the client, how can we prevent the attackers from eavesdropping on the message? The server can attach its signature to it, and the client can accept the message only if the verification of the certificate passes. So this process can be divided into two parts: `sign the message to get the digital signature` and `verify the digital signature`.

- **Sign the message**: means `hash` the message and `encrypt` the hash with the server's private key. The encrypted hash is called `digital signature`. 

- **Verify the signature**: means `decrypt` the signature with the server's public key to get the hash, `re-compute` another hash of the message, and compare the identity of two hashes.

What can we benefit from signing and verifying the digital signature? To verify the digital signature is to confirm two things: 

- **Integrity**: the message has not changed since the signature was attached because it is based on a  `cryptographic hash` of the message. 

- **Authentication**: the signature belongs to the person who alone has access to the private key. 

Information security has other attributes, but integrity and authentication are the two traits you must know.

### Secrets behind digital signature

As mentioned in the above section, the digital signature can prove the integrity of the message and the authenticity of the message's owner. To understand how it works, you must understand the following two facts: 

- **Hash is irreversible**

In the previous articles, we explained that cryptography is a two-way algorithm. You can `encrypt` the plaintext to the ciphertext and `decrypt` the ciphertext back to the plaintext. It means the cryptographic algorithm is reversible. 

Different from cryptographic algorithms, **A hash is irreversible!**. This irreversibility is the whole point.

The goal of any cryptographic hashing algorithm is to reduce the arbitrarily sized input into a fixed-sized hash. The cryptographic hashing algorithm should guarantee that no two different messages can produce an identical hash. That is `collide`. 

In this way, it is impossible for attackers to reverse engineer such a `collision`. 

<img src="/images/https-signature-hash.png" title="Hash is irreversible" width="600px" height="400px">

The attacker intercepts the message and changes it. Then the verification process of the signature on the client-side can not pass since the re-computed hashing can not match the original hashing. 

I will write about how to implement a hash function in the future. In this article, you can ignore the details. Just remember that hashing is irreversible. 

**The private key prove identity**

The attack shown above does not pass the verification because the signature and the forged message do not match. Then this time, the attacker compute a new hash based on the forged message and sign the hash with his private key. Can this attack work? 

<img src="/images/https-signature-key-identity.png" title="Hash is irreversible" width="800px" height="600px">

The short answer is No. 

When the client gets the digital signature, the first is to `decrypt` it. The decryption breaks for this new attack because the forged signature is signed with the attacker's private key instead of the server's private key. 

**It’s impossible for anybody except the owner of the private key to generate something that can be decrypted using the public key.** This is the nature of public-key cryptography. I will write other articles to explain it mathematically in the future.  

In a word, the private key can prove identity. 

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


