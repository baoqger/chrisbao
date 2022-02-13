---
title: "How HTTPS works: part five - manual verification of SSL/TLS certificates"
date: 2019-3-6 16:00:47
tags: digital signature, hash function
---
### Background

In the previous articles of this series, we examined how digital signatures and certificates make the `TLS handshake` process work. Generally speaking, this process is handled by platforms like web browsers. In this article, I want to **demystify the process of TLS handshake by manual verification of SSL/TLS certificates step by step**. 

### Certificate chain

In this previous [article](http://localhost:4000/2019/03/02/https-certificate-anatomy/), we mentioned that **the server returns not one, but multiple certificates. These associated certificates form a certificate chain** as follows: 

<img src="/images/https-certificates-chain.png" title="certificates chain" width="800px" height="600px">

The intermediate CA signs the server certificate, and Root CA signs the certificate of the intermediate CA (There can be multiple intermediate CAs). Pay attention to the `subject` and `issuer` names of each certificate, and you can notice that one certificate's issue name is another certificate's subject name. Certificates can find their CA's certificate in the chain with this information. 

For example, the following `openssl` command prints the certificate chain of google.com: 

```shell
openssl s_client -showcerts -connect google.com:443
```

The previous [article](http://localhost:4000/2019/03/02/https-certificate-anatomy/) explained that the chain contains three certificates. Let us extract the google.com server certificate into file [**google_com.pem**](https://gist.github.com/baoqger/8c854336118737db2cb55997ca7888c9) and the intermediate CA certificate into file [**intermediate_ca.pem**](https://gist.github.com/baoqger/59cd171e2114030a9f70790e99bd86d0). These two files are the input data for this manual verification experiment. 

### Manual verification of SSL/TLS certificates

The entire process can be illustrated as follows: 

<img src="/images/https-manual-verification.png" title="certificates chain" width="800px" height="600px">

Let us go through it step by step. 

#### Extract the public key of intermediate CA

Since the certificate of the google.com server is signed with intermediate CA's private key. The first step is to extract the public key of intermediate CA from its certificate. 

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
You can look at the public key in PEM format (which is not readable to humans). 

#### Extract the signature

Next, let us extract the digital signature of the google.com server's certificate. This task is a little bit complex, and we need several Linux commands to form a data stream pipeline as follows: 

```shell
openssl x509 -in ./google_com.pem -text -noout -certopt ca_default,no_validity,no_serial,no_subject,no_extensions,no_signame \
| grep -v 'Signature Algorithm' \
| tr -d '[:space:]:' \
| xxd -r -p > ./certificate-signature.bin
```
- `openssl x509`: extract the digital signature. `openssl` command supports `certopt` option, which allows multiple arguments separated by commas. The output of the first command should be:

```
Signature Algorithm: sha256WithRSAEncryption
        76:6a:8a:d8:44:ac:14:40:92:36:f3:4a:b5:cb:54:36:67:c7:
        <... content omitted ...>
        14:22:3f:2a:90:a5:e4:9b:26:df:33:15:4b:d2:5c:f7:89:8e:
        f7:6a:c4:a6
```

- `grep -v`: invert the sense of matching and select non-matching lines (because of -v option).`Signature Algorithm: sha256WithRSAEncryption` is matched, but only the Hex code lines are selected based on the invert-match rule. The data stream after filtering should be:

```
        76:6a:8a:d8:44:ac:14:40:92:36:f3:4a:b5:cb:54:36:67:c7:
        <... content omitted ...>
        14:22:3f:2a:90:a5:e4:9b:26:df:33:15:4b:d2:5c:f7:89:8e:
        f7:6a:c4:a6
```

Note **Signature Algorithm: sha256WithRSAEncryption** means that the signature is signed with `SHA-256` hash function and `RSA` encryption algorithm. When we need to re-compute the hash again, we must use the same hash function: `SHA-256`. 

- `tr -d`: delete the colons. The colons are to make it more readable. After removing the formatting colons, the output only contains Hex code. 

- `xxd -r -p`: convert the Hex code into binary format and redirect to file **certificate-signature.bin** 

#### Decrypt the signature

Now that we have both the digital signature and the public key of the intermediate CA. We can decrypt the signature as follows:

```shell 
openssl rsautl -verify -inkey ./intermediate_ca_pub.pem -in ./certificate-signature.bin -pubin > ./certificate-signature-decrypted.bin
```

Store the decrypted signature in file **certificate-signature-decrypted.bin**. We can view the hash with openssl like so:

```shell
openssl asn1parse -inform der -in ./certificate-signature-decrypted.bin
```
The output is: 
```
0:d=0  hl=2 l=  49 cons: SEQUENCE
2:d=1  hl=2 l=  13 cons: SEQUENCE
4:d=2  hl=2 l=   9 prim: OBJECT            :sha256
15:d=2  hl=2 l=   0 prim: NULL
17:d=1  hl=2 l=  32 prim: OCTET STRING      [HEX DUMP]:66EFBE4CEA76272C76CEE8FA297C1BF70C41F8E049C7E0E4D23C965CBE8F1B84
```
The hash is in the last line: `66EFBE4CEA76272C76CEE8FA297C1BF70C41F8E049C7E0E4D23C965CBE8F1B84`. 

#### Re-compute the hash

Now that we got the original hash of the certificate, we need to verify whether we can re-compute the same hash using the same hashing function (as mentioned above SHA-256 in this case).

The previous [article](https://organicprogrammer.com/2019/03/05/https-certificate-signature/) explained that **the digital signature is signed based on all of the information instead of only public key**. We need to extract everything but the signature from the certificate. 

We can extract the data and output it to file **google_com_cert_body.bin** as follows: 

```
openssl asn1parse -i -in ./google_com.pem -strparse 4 -out ./google_com_cert_body.bin  -noout
```

Note: you can refer to this openssl [document](https://linux.die.net/man/1/asn1parse) to have a deeper understanding about the behavior of this command. 

Finally, let us re-compute the hash with SHA256 as follows: 

```shell
openssl dgst -sha256 ./google_com_cert_body.bin

SHA256(./google_com_cert_body.bin)= 66efbe4cea76272c76cee8fa297c1bf70c41f8e049c7e0e4d23c965cbe8f1b84
```

The re-computed hash is `66efbe4cea76272c76cee8fa297c1bf70c41f8e049c7e0e4d23c965cbe8f1b84`, which matches with the original hash. So we can get the conclusion that: **the intermediate CA signs the goole.com server certificate**. 

### Summary

In this article, we go through the verification process of SSL/TLS certificates step by step manually. I have to admit that the concepts of digital signatures and certificates are too abstract to understand, especially in the details. I hope the experiment, which carries out with certificates from the real world, can help you do that.  


