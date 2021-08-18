---
title: How to write a Golang HTTP server with Linux system calls 
date: 2021-07-31 10:15:25
tags: HTTP, system call, Linux, Golang
---

### Background

`HTTP` is everywhere. As a software engineer, you're using the `HTTP` protocol every day. Starting an `HTTP` server will be an easy task if you're using any modern language or framework. For example, in Golang you can do that with the following lines of code:

```golang
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/hi", func(w http.ResponseWriter, r *http.Request){
        fmt.Fprintf(w, "Hi")
    })

    log.Fatal(http.ListenAndServe(":8081", nil))
}
```

You can finish the job easily because `net/http` package implements the `HTTP` protocol completely. Without `net/http` package how can you do that? That's the target of this article. 

**Note**: This article is inspired by [Joe Schafer's post](https://joe.schafer.dev/go-server-with-syscalls/) a lot. My implementation has something different which totally removes dependency on Golang's `net` package, but the idea of using `system call` in Golang to setup the TCP/IP connetion is the same. Thanks very much for Joe Schafer's interesting post.

Another thing I need to mention is this article will cover many concepts, but it's very difficult to discuss all of them in the detail. To understand this article smoothly, you need some prerequisite knowledge such as `OSI model`, `TCP/IP protocol`, `socket programming`, `HTTP protocol` and `system call`. I will add some explanations on these topics to help you understand this article and give some references and links to let you continue exploring more in advanced level. 

### OSI model, TCP/IP and HTTP

##### OSI network model

[`OSI model`](https://en.wikipedia.org/wiki/OSI_model) partitions the data flow in a communication system into **seven abstraction layers**. These layers form a protocol stack, with each layer communicating with the layer above and the layer below as follows: 

<img src="/images/osi_model.png" title="network" width="400px" height="300px">

For example, `HTTP` is in **layer 7**, `TCP` is in **layer 4** and `IP` is in **layer 3**. 

OSI is a general model, which can be simplified into a layered model more  consistent with Unix as follows: 

- Application Layer (telnet, ftp, http)
- Host-to-Host Transport Layer (TCP, UDP)
- Internet Layer (IP and routing)
- Network Access Layer (Ethernet, wi-fi)

The critical point to understand is `data encapsulation`.  The data flow goes from the bottom physical level to the highest-level represention of data in  an application.

Each layer has administrative information that it has to keep about its own layer. It does this by adding header information to the packet it receives from the layer above, as the packet passes down. Each header contains information regarding the message contents. For example, the `HTTP` sends data from one host to another. It uses the `TCP` protocol on top of the `IP` protocol, which may be sent over `Ethernet`. This looks like: 

<img src="/images/data_encapsulation.png" title="network" width="400px" height="300px">



The packet transmitted over ethernet, is the bottom one. On the receiving side, these headers are removed as the packet moves up.



##### TCP/IP

<img src="/images/socket_network.png" title="network" width="600px" height="400px">

##### Socket

##### HTTP

* reference to other articles. difference to other articles
* basic OSI model and TCP/IP socket network programming
* HTTP message structure
* socket programming with Linux system call: create new socket, bind, listen. Read and writ to or from the socket
* what is socket? file descriptor
* HTTP parser. SimpleNet package


