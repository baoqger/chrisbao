---
title: "How HTTPS works: part three - the anatomy of certificate"
date: 2019-3-2 16:00:47
tags: certificate, digital signature, hashing
---

### Background

In the [last article](https://organicprogrammer.com/2019/02/25/https-certificate/), we examined the importance of certificates, which prevent us from the man-in-the-middle attacks. But we did not explain the mechanics of certificates, and it will be the focus of the following articles. In this article, let us first examine how the `TLS certificate` looks like and what kind of information it contains? 

### Anatomy of certificate

The official name of `SSL/TLS certificate` is [`X.509 certificate`](https://en.wikipedia.org/wiki/X.509). Before we can examine the content of the certificate, we must get it. We can do this with `openssl`. `openssl` is a popular tool in the network security field. In the following articles, we will use it a lot. For the use of `openssl`, you can refer to this online [document](https://www.feistyduck.com/books/openssl-cookbook/). 

Let us pull the certificate from google.com domain as follows: 

```shell
openssl s_client -showcerts -connect google.com:443
```
The output contains a large amount of information, and I omit some unnecessary codes to make it compact as follows: 
```
CONNECTED(00000003)
depth=2 C = US, O = Google Trust Services LLC, CN = GTS Root R1
verify return:1
depth=1 C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
verify return:1
depth=0 CN = *.google.com
verify return:1
---
Certificate chain
 0 s:CN = *.google.com
   i:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
-----BEGIN CERTIFICATE-----
MIINsjCCDJqgAwIBAgIRAPq8ife/MxCUCgAAAAEl/TIwDQYJKoZIhvcNAQELBQAw
FYN5klRqI0hWa3wYe6tnXm/2PvPbAwqsnAq3q+Iek+3pGm6YTshJyA7P9L176psd
<... content omitted ...>
dm6slAYpHOryFcrvXzu1lHSylCAFNT/OYcH1GLTf0qJXuN7YnX9swoYu2oCDkIyA
Hss2DDp7f8qf0VgDNNxZB8drZ9ID85YA3qgeIbHHAB8UIj8qkKXkmybfMxVL0lz3
iY73asSm
-----END CERTIFICATE-----
 1 s:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R1
-----BEGIN CERTIFICATE-----
MIIFljCCA36gAwIBAgINAgO8U1lrNMcY9QFQZjANBgkqhkiG9w0BAQsFADBHMQsw
CQYDVQQGEwJVUzEiMCAGA1UEChMZR29vZ2xlIFRydXN0IFNlcnZpY2VzIExMQzEU
<... content omitted ...>
AJ2xDx8hcFH1mt0G/FX0Kw4zd8NLQsLxdxP8c4CU6x+7Nz/OAipmsHMdMqUybDKw
juDEI/9bfU1lcKwrmz3O2+BtjjKAvpafkmO8l7tdufThcV4q5O8DIrGKZTqPwJNl
1IXNDw9bg1kWRxYtnCQ6yICmJhSFm/Y3m6xv+cXDBlHz4n/FsRC6UfTd
-----END CERTIFICATE-----
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R1
   i:C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
-----BEGIN CERTIFICATE-----
MIIFYjCCBEqgAwIBAgIQd70NbNs2+RrqIQ/E8FjTDTANBgkqhkiG9w0BAQsFADBX
MQswCQYDVQQGEwJCRTEZMBcGA1UEChMQR2xvYmFsU2lnbiBudi1zYTEQMA4GA1UE
<... content omitted ...>
9U5pCZEt4Wi4wStz6dTZ/CLANx8LZh1J7QJVj2fhMtfTJr9w4z30Z209fOU0iOMy
+qduBmpvvYuR7hZL6Dupszfnw0Skfths18dG9ZKb59UhvmaSGZRVbNQpsg3BZlvi
d0lIKO2d1xozclOzgjXPYovJJIultzkMu34qQb9Sz/yilrbCgj8=
-----END CERTIFICATE-----
---
Server certificate
subject=CN = *.google.com

issuer=C = US, O = Google Trust Services LLC, CN = GTS CA 1C3

---
<... content omitted ...>
```

Note: if you want to take a look at the full content of the above output, please refer to this online [gist file](https://gist.github.com/baoqger/d1d5792d17c4b260ca186d9d2651066b)

Based on the format of the output, you can figure out that the content between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` markers is a certificate. 

Another remarkable point is that the server returns not one, but three certificates. These associated certificates form a `certificate chain`. I will examine it in future articles. In the current post, let us focus on the certificate itself. 

As shown above, the default format of the certificate is `PEM(Privacy Enhanced Mail)`. PEM is a Base64 encoded binary format. We can convert it to a human-readable format with `openssl`.

First, let us extract the google.com server certificate into a file named **google_com.pem**. Remember to include the `BEGIN` and `END` markers, but nothing more. You can refer to this [file](https://gist.github.com/baoqger/8c854336118737db2cb55997ca7888c9).

Then, run the following openssl command: 

```
openssl x509 -in google_com.pem -noout -text
```
here is the [output](https://gist.github.com/baoqger/79923d4f92b166bb914aa700721d6a0b) (with some content omitted):

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            fa:bc:89:f7:bf:33:10:94:0a:00:00:00:01:25:fd:32
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
        Validity
            Not Before: Nov 29 02:22:33 2021 GMT
            Not After : Feb 21 02:22:32 2022 GMT
        Subject: CN = *.google.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:a1:c2:d2:74:cc:32:68:86:68:18:0c:8f:5a:e1:
                    <... content omitted ...>
                    c8:1b:6b:85:5d
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                04:0E:70:9D:60:11:97:01:1B:0E:7E:72:B1:99:E5:28:F7:29:E0:72
            X509v3 Authority Key Identifier:
                keyid:8A:74:7F:AF:85:CD:EE:95:CD:3D:9C:D0:E2:46:14:F3:71:35:1D:27

            Authority Information Access:
                OCSP - URI:http://ocsp.pki.goog/gts1c3
                CA Issuers - URI:http://pki.goog/repo/certs/gts1c3.der

            X509v3 Subject Alternative Name:
                DNS:*.google.com, DNS:*.appengine.google.com, DNS:*.bdn.dev, DNS:*.cloud.google.com, DNS:*.crowdsource.google.com, DNS:*.datacompute.google.com, DNS:*.google.ca, DNS:*.
                <... content omitted ...>
                youtubekids.com, DNS:yt.be, DNS:*.yt.be, DNS:android.clients.google.com, DNS:developer.android.google.cn, DNS:developers.android.google.cn, DNS:source.android.google.cn
            X509v3 Certificate Policies:
                Policy: 2.23.140.1.2.1
                Policy: 1.3.6.1.4.1.11129.2.5.3

            X509v3 CRL Distribution Points:

                Full Name:
                  URI:http://crls.pki.goog/gts1c3/QqFxbi9M48c.crl

            CT Precertificate SCTs:
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 29:79:BE:F0:9E:39:39:21:F0:56:73:9F:63:A5:77:E5:
                                BE:57:7D:9C:60:0A:F8:F9:4D:5D:26:5C:25:5D:C7:84
                    Timestamp : Nov 29 03:22:38.708 2021 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:46:02:21:00:B5:C7:D9:40:A7:62:19:B6:D8:62:D2:
                                <... content omitted ...>
                                F9:53:FA:C7:EF:2C:BA:9C
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 41:C8:CA:B1:DF:22:46:4A:10:C6:A1:3A:09:42:87:5E:
                                4E:31:8B:1B:03:EB:EB:4B:C7:68:F0:90:62:96:06:F6
                    Timestamp : Nov 29 03:22:38.872 2021 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:87:35:02:A8:F6:06:EF:BC:F4:C1:95:
                                <... content omitted ...>
                                7E:5B:8B:35:E6:D2:3C
    Signature Algorithm: sha256WithRSAEncryption
         76:6a:8a:d8:44:ac:14:40:92:36:f3:4a:b5:cb:54:36:67:c7:
         3a:a5:e9:b5:31:6c:51:5f:f3:ed:6a:99:ac:a7:5b:9c:ae:c9:
         <... content omitted ...>
         59:07:c7:6b:67:d2:03:f3:96:00:de:a8:1e:21:b1:c7:00:1f:
         14:22:3f:2a:90:a5:e4:9b:26:df:33:15:4b:d2:5c:f7:89:8e:
         f7:6a:c4:a6
```
The certificate contains many elements. They are specified using a syntax
referred to as `ASN.1(Abstract Syntax Notation)`. As you can see, the certificate consists of two parts: **Data** and **Signature Algorithm**. The **Data** part contains the identity information about this server certificate. And the second part is the `digital signature` of this certificate. I'll highlight the following elements:

- **Issuer**: identifies who issues this certificate, also known as CA(certificate authority).
- **Subject**: identifies to whom the certificate is issued. In the case of the server certificate, the `Subject` should be the organization that owns the server. But in the certificate chain, the `Issuer` of the server certificate is the `Subject` of the intermediate certificate. I'll examine this relationship in detail in the following article. 
- **Subject Public Key Info**: the server's public key. As we mentioned above, we created the certificate just to send the public key to the client. 
- **Signature Algorithm**: refers to the element at the bottom of the certificate. The **sha256WithRSAEncryption** part denotes the `hash function` and `encryption algorithm` used to `sign the certificate`. In the above case, the server uses `sha256` as the hash function and `RSA` as the encryption algorithm. And the following block contains the signed hash of X.509 certificate data, called `digital signature`. The digital signature is the key information in certificates to establish trust between clients and servers. I'll explain how a digital certificate works in the next article. 

### Summary

This article examines the information contained inside certificates using the tool `openssl`. We highlight some critical elements, and in the following article let us take a deep look at `digital signature`.
