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

- **Verify the signature**: means `decrypt` the signature with the server's public key to get the hash, `re-compute` the other hash of the message, and compare the identity of two hashes.

What can we benefit from signing and verifying the digital signature? To verify the digital signature is to confirm two things: 

- **Message Integrity**: the message has not changed since the signature was attached because it is based on a  `cryptographic hash` of the message. 

- **Proof of Origin**: the signature belongs to the person who alone has access to the private key. The recipient of the message must be sure of the origin of the message.

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

### How does signature make certificate secure?

The digital signature is not designed only for TLS certificates, and it has other applications, which is not the focus of my article. The important thing is to understand how to apply digital signature in the PKI framework and how it makes certificates more secure. 

The key is that **the certificate associates the public key with the server you are connecting to**. As we see in [last](https://organicprogrammer.com/2019/03/02/https-certificate-anatomy/) article, the certificate must contain some information about the identity of the server, such as the domain name of the server. For example, the `Subject` property of the [certificate](https://gist.github.com/baoqger/79923d4f92b166bb914aa700721d6a0b) we examined in the last article

```
Subject: CN = *.google.com
```

And the digital signature is `signed based on all of the information instead of only public key`. It can prevent man-in-the-middle attack in the two following cases:

- The attacker intercepts the server's certificate and changes the public key or any other information, the hash code in the signature does not match the hash code of the content of the forged certificate, and the client rejects it. It is `Message Integrity` mentioned above. 

- The attacker can obtain a certificate signed by the trusted CA by pretending as a legitimate business. When the client requests a certificate from the server, the attacker can replace it with his own. The validation on the client-side can not pass. Although the attacker's certificate is signed by a trusted CA, the domain name(or other server information) does not match the expected one. It is `Proof of Origin` mentioned above. 

So far, I hope you can understand (roughly) the beauty of this Internet security framework. 

### Summary

In this article, I examined how a digital signature works. You see how to sign a signature and how to verify it. Digital signature and certificate are the most abstract part of the PKI framework. So in the following article, let me illustrate the whole process by manually going through the certificate verification process step by step. We will use `openssl` to examine the result in each step.


