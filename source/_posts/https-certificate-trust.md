---
title: https-certificate-trust
date: 2022-01-04 16:35:35
tags:
---


4 certificate的内容
PEM format
用openssl convert to plain text
大致分析它的内容

５　数值签名（值得单独一篇文章）
How are digital signatures used and why are they important? Digital signatures are how certificates are authenticated and how you can determine whether or not to trust a certificate. 

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

