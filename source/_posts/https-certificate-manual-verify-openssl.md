---
title: "How HTTPS works: part five - manual verification of SSL/TLS certificates"
date: 2019-3-6 16:00:47
tags: digital signature, hash function
---
### Background

In the previous articles of this series, we examined how digital signature and certificate make `TLS handshake` process work. Generally speaking, this  process is handled by platforms like web browsers. In this article, I want to **demystify the process of TLS handshake by manual verification of SSL/TLS certificates step by step**. 

### Certificate chain

In this previous [article](http://localhost:4000/2019/03/02/https-certificate-anatomy/), we mentioned that **the server returns not one, but multiple certificates. These associated certificates form a certificate chain** as follows: 

<img src="/images/https-certificates-chain.png" title="certificates chain" width="800px" height="600px">

The server's certificate is signed by an intermediate CA, and the certificate of the intermediate CA is signed by Root CA (There can be multiple intermediate CAs).Take a loot at the `subject` and `issuer` name of each certificate, you can notice that one certificate's issue name is another certificate's subject name. Certificates can find its CA's certificate with this information. 

For example, the following `openssl` command will print certificate chain of google.com: 

```shell
openssl s_client -showcerts -connect google.com:443
```

As we explained in the previous [article](http://localhost:4000/2019/03/02/https-certificate-anatomy/), the chain contains three certificates. Let us extract the google.com server certificate into file [**google_com.pem**](https://gist.github.com/baoqger/8c854336118737db2cb55997ca7888c9) and the intermediate CA certificate into file [**intermediate_ca.pem**](https://gist.github.com/baoqger/59cd171e2114030a9f70790e99bd86d0). These two files are the input data for this manual verification experiment. 

### Manual verification of SSL/TLS certificates

The entire process can be illustrated as follows: 

<img src="/images/https-manual-verification.png" title="certificates chain" width="800px" height="600px">

Let us go through step by step. 

#### Extract the public key of intermediate CA

Since the certificate of google.com server is signed with intermediate CA's private key. The first step we need to do is extracting the public key of intermediate CA from its certificate. 

```shell
openssl x509 -in ./intermediate_ca.pem -noout -pubkey > ./intermediate_ca_pub.pem
```

The content of file **intermediate_ca_pub.pem** goes as follows:

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA9Yjf52KMHjf4N0KQf2yH
0PtlgiX96MtrpP9t6Voj4pn2HOmSA5kTfAkKivpC1l5WJKp6M4Qf0elpu7l07FdM
ZmiTdzdVU/45EE23NLtfJXc3OxeU6jzlndW8w7RD6y6nR++wRBFj2LRBhd1BMEiT
G7+39uBFAiHglkIXz9krZVY0ByYEDaj9fcou7+pIfDdNPwCfg9/vdYQueVdc/Fdu
Gpb//Iyappm+Jdl/liwG9xEqAoCA62MYPFBJh+WKyl8ZK1mWgQCg+1HbyncLC8mW
T+9wScdcbSD9mbS04soud/0t3Au2axMMjBkrF5aYufCL9qAnu7bjjVGPva7Hm7GJ
nQIDAQAB
-----END PUBLIC KEY-----
```
You can take a look at the public key content in PEM format (which is not readable to humans). 

#### Extract the signature

Next, let us extract the digital signature of google.com server's certificate. This task is a little bit complex, and we need several Linux commands to form a data stream pipeline as follows: 

```shell
openssl x509 -in ./google_com.pem -text -noout -certopt ca_default,no_validity,no_serial,no_subject,no_extensions,no_signame \
| grep -v 'Signature Algorithm' \
| tr -d '[:space:]:' \
| xxd -r -p > ./certificate-signature.bin
```
- `openssl x509`: extract the digital signature. `openssl` supports `certopt` option which allows multiple arguments separated by commas. The output of the first command should be:

```
Signature Algorithm: sha256WithRSAEncryption
        76:6a:8a:d8:44:ac:14:40:92:36:f3:4a:b5:cb:54:36:67:c7:
        <... content omitted ...>
        14:22:3f:2a:90:a5:e4:9b:26:df:33:15:4b:d2:5c:f7:89:8e:
        f7:6a:c4:a6
```

- `grep -v`: invert the sense of matching and select non-matching lines (because of -v option). The first line `Signature Algorithm: sha256WithRSAEncryption` is matched but based on the invert match rule only the Hex code lines will be selected. The data stream after filtering should be:

```
        76:6a:8a:d8:44:ac:14:40:92:36:f3:4a:b5:cb:54:36:67:c7:
        <... content omitted ...>
        14:22:3f:2a:90:a5:e4:9b:26:df:33:15:4b:d2:5c:f7:89:8e:
        f7:6a:c4:a6
```

Note **Signature Algorithm: sha256WithRSAEncryption** means that the signature is signed with `SHA-256` hash function and `RSA` encryption algorithm. In the following step, we need to re-compute the hash with the same hash function `SHA-256`. 

- `tr -d`: delete the colons. The colons are just to make it more readable. After removing the formatting colons, the output is just Hex code. 

- `xxd -r -p`: convert the Hex code into binary format and redirect to file **certificate-signature.bin** 



思路

certificates chain
配合插图大致解释chain的意思，一个certificate的issuer是下一个certificate的subject
不要解释为什么需要chain的结构，放到以后专门解释, 这涉及trust架构问题

一步一步介绍verfication的过程

下一篇文章：
如何hack这个系统呢？结合联想的那个事件分析下

