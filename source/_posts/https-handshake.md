---
title: https-handshake
date: 2021-12-22 11:04:58
tags:
---

outline

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
