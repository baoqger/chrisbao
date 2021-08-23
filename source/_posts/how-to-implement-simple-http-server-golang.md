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

Another thing I need to mention is this article will cover many concepts, but it's very difficult to discuss all of them in the detail. To understand this article smoothly, you need some prerequisite knowledge such as `OSI model`, `TCP/IP stack`, `socket programming`, `HTTP protocol` and `system call`. I will add some explanations on these topics to help you understand this article and give some references and links to let you continue exploring more in advanced level. 

### OSI network model

[`OSI model`](https://en.wikipedia.org/wiki/OSI_model) partitions the data flow in a communication system into **seven abstraction layers**. These layers form a protocol stack, with each layer communicating with the layer above and the layer below as follows: 

<img src="/images/osi_model.png" title="network" width="400px" height="300px">

For example, `HTTP` is in **layer 7**, `TCP` is in **layer 4** and `IP` is in **layer 3**. 

OSI is a general model, which was first specified in the early 1980s. **But neither traditional nor modern networking protocols fit into this model neatly**. For example, `TCP/IP` stack does not define the three upper layers: session, presentation, and application. In fact, it does not define anything above the transport layer. From the viewpoint of `TCP/IP`, everything above the transport layer is part of
the application. So the layered network model more  consistent with Linux (TCP/IP stack is implemented in Linux kernel) is as follows: 

- Application Layer (telnet, ftp, http)
- Host-to-Host Transport Layer (TCP, UDP)
- Internet Layer (IP and routing)
- Network Access Layer (Ethernet, wi-fi)

Once again, **it is important to point out that the upper layers—Layers 5, 6, and 7—are not part of the TCP/IP stack**. 

Another critical point to understand is `data encapsulation`.  The data flow goes from the bottom physical level to the highest-level representation of data in an application.

Each layer has administrative information that it has to keep about its own layer. It does this by adding header information to the packet it receives from the layer above, as the packet passes down. Each header contains information regarding the message contents. For example, the `HTTP` sends data from one host to another. It uses the `TCP` protocol on top of the `IP` protocol, which may be sent over `Ethernet`. This looks like: 

<img src="/images/data_encapsulation.png" title="network" width="400px" height="300px">



The packet transmitted over ethernet, is the bottom one. On the receiving side, these headers are removed as the packet moves up.

Next let's see how `TCP/IP` stack encapsulates `HTTP` message and send it over the network through `socket`. The idea can be illustrated with the following image: 

<img src="/images/socket_network.png" title="network" width="600px" height="400px">

I will explain how it works by writing a HTTP server from scratch, you can refer to this [Github repo](https://github.com/baoqger/http-server-scratch) to get all the code. 

### TCP/IP

`TCP/IP` stack is originated from [`ARPANET`](https://en.wikipedia.org/wiki/ARPANET) project, which is integrated into Unix BSD OS as the first implementation of `TCP/IP` protocols.

Nowadays, `TCP/IP` is still implemented in the operating system level. For Linux system, you can find the source code inside the kernel. The detailed implementation is outside the scope of this article. You can study it in this Github [link](https://github.com/torvalds/linux/tree/master/net/ipv4). 

### Socket

As I mentioned in the above sections, HTTP server is a running in the application level. How it can work with `TCP/IP` stack which lives in the kernel? The answer is `socket`. 

The `socket` interface was originally developed as part of the BSD operating system. Sockets provide an interface between the application level programs and the TCP/IP stack. Linux (or other OS) provides an API and sockets, and applications use this API to access the networking facilities in the kernel. 

The socket interface is really TCP/IP's window on the world. In most modern systems incorporating TCP/IP, the socket interface is the only way that applications make use of the TCP/IP suite of protocols. 

One main advantage of sockets in Unix or Linux system is that the socket is treated as a `file descriptor`, and all the standard I/O functions work on sockets in the same way they work on a local file. File descriptor is simply an integer associated with an open file. 

You may heard **everything in Unix is a file**. The file can be a network connection, a pipe, a real file in the disk, a device or anything else. So when you want to send data to another program over the Interent you will do it through a file descriptor.   

In our HTTP server case, **it will get the request by reading data from the socket and send the response by writing data to the socket**.  

Next, let's review the source code to see how the HTTP server is implemented.

First, we need setup the TCP connection through socket, the process can be described in the following image: 

<img src="/images/socket_tcp.png" title="network" width="600px" height="400px">

In Golang, `net` package provides all the socket related functionalities. Since this article's purpose is writing a HTTP server from scratch, so I create a package named **simplenet** to provide the very basic implementation. 

```golang
package simplenet

import (
	"os"
	"syscall"
)

type netSocket struct {
	fd int
}

func NewNetSocket(ip IP, port int) (*netSocket, error) {
	// ForkLock docs state that socket syscall requires the lock.
	syscall.ForkLock.Lock()
	// AF_INET = Address Family for IPv4
	// SOCK_STREAM = virtual circuit service
	// 0: the protocol for SOCK_STREAM, there's only 1.
	fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
	if err != nil {
		return nil, os.NewSyscallError("socket", err)
	}
	syscall.ForkLock.Unlock()

	// Allow reuse of recently-used addresses.
	if err = syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1); err != nil {
		syscall.Close(fd)
		return nil, os.NewSyscallError("setsockopt", err)
	}

	// Bind the socket to a port
	sa := &syscall.SockaddrInet4{Port: port}
	copy(sa.Addr[:], ip)
	if err = syscall.Bind(fd, sa); err != nil {
		return nil, os.NewSyscallError("bind", err)
	}

	// Listen for incoming connections.
	if err = syscall.Listen(fd, syscall.SOMAXCONN); err != nil {
		return nil, os.NewSyscallError("listen", err)
	}

	return &netSocket{fd: fd}, nil
}

func (ns netSocket) Read(p []byte) (int, error) {
	if len(p) == 0 {
		return 0, nil
	}
	n, err := syscall.Read(ns.fd, p)
	if err != nil {
		n = 0
	}
	return n, err
}

func (ns netSocket) Write(p []byte) (int, error) {
	n, err := syscall.Write(ns.fd, p)
	if err != nil {
		n = 0
	}
	return n, err
}

// Creates a new netSocket for the next pending connection request.
func (ns *netSocket) Accept() (*netSocket, error) {
	// syscall.ForkLock doc states lock not needed for blocking accept.
	nfd, _, err := syscall.Accept(ns.fd)
	if err == nil {
		syscall.CloseOnExec(nfd)
	}
	if err != nil {
		return nil, err
	}
	return &netSocket{nfd}, nil
}

func (ns *netSocket) Close() error {
	return syscall.Close(ns.fd)
}
```

**netSocket** data model is created to represent the socket, which contains only one field **fd** means file descriptor. And all the socket related APIs: **Read**, **Write**,  **Accept** and **Close**, are defined. 

The logic of **netSocket** is not complex, because it delegates the job to the kernel by `system call`. A system call is a programmatic way a program requests a service from the kernel, in detail you can refer to this [article](https://opensource.com/article/19/10/strace). In Golang, all the system calls are wrapped inside the `syscall` standard package.  

One thing need to mention is different platform have different `syscall` usages, so the demo code shown in this article can only be compiled and build on Linux system. 

### HTTP

* reference to other articles. difference to other articles
* basic OSI model and TCP/IP socket network programming
* HTTP message structure
* socket programming with Linux system call: create new socket, bind, listen. Read and writ to or from the socket
* what is socket? file descriptor
* HTTP parser. SimpleNet package


