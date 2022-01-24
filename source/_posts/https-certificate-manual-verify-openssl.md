---
title: https-certificate-manual-verify-openssl
date: 2019-3-6 16:00:47
tags:
---




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
