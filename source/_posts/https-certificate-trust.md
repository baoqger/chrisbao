---
title: "How HTTPS works: part three - the anatomy of certificate"
date: 2019-3-2 16:00:47
tags: certificate, digital signature, hashing
---

### Background

In the [last article](https://organicprogrammer.com/2019/02/25/https-certificate/), we examined the importance of certificates, which can prevent us from the man-in-the-middle attacks. But we didn't explain the mechanics of certificates, and that will be the follow articles' focus. In this article, let's first take a look at how the `TLS certificate` look like and what kind of information it contains? 

### Anatomy of certificate

思路：书接上文，certificate可以防止man-in-the-middle攻击。那么它是如何做到的呢？其中的奥妙是什么呢？
得用后面多篇文章来解释这个原理。
第一篇文章介绍certificate的内容(通过openssl获取,并解析它的内容)。引出digital signature

第二篇介绍 digital signature

(中间有个certificates chain的概念，应该放到哪里🤔)

第三篇介绍 手动验证signature

第四篇介绍 trusts/certificates chain

金句：
The certificate encodes two very important pieces of information: the server's public key and a digital signature that can be used to confirm the certificate's authenticity.  

