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

You can finish the job easily because `net/http` package implements the `HTTP` protocol completely. How can you do that without `net/http` package ? That's the target of this article. 

**Note**: This article is inspired by [Joe Schafer's post](https://joe.schafer.dev/go-server-with-syscalls/) a lot. My implementation has something different which totally removes dependency on Golang's `net` package, but the idea of using `system call` in Golang to setup the TCP/IP connetion is the same. Thanks very much for Joe Schafer's interesting post.

Another thing I need to mention is this article will cover many concepts, but it's very difficult to discuss all of them in detail. To understand this article smoothly, you need some prerequisite knowledge such as `OSI model`, `TCP/IP stack`, `socket programming`, `HTTP protocol` and `system call`. I will add some explanations on these topics to help you understand this article and give some references and links to let you continue exploring more in advanced level. 

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

Each layer has administrative information that it has to keep about its own layer. It does this by adding header information to the packet it receives from the layer above, as the packet passes down. Each header contains information regarding the message contents. For example, one `HTTP` server sends data from one host to another. It uses the `TCP` protocol on top of the `IP` protocol, which may be sent over `Ethernet`. This looks like: 

<img src="/images/data_encapsulation.png" title="network" width="400px" height="300px">



The packet transmitted over ethernet, is the bottom one. On the receiving side, these headers are removed as the packet moves up.

Next let's see how `TCP/IP` stack encapsulates `HTTP` message and send it over the network through `socket`. The idea can be illustrated with the following image: 

<img src="/images/socket_network.png" title="network" width="600px" height="400px">

I will explain how it works by writing a HTTP server from scratch, you can refer to this [Github repo](https://github.com/baoqger/http-server-scratch) to get all the code. 

### TCP/IP

`TCP/IP` stack is originated from [`ARPANET`](https://en.wikipedia.org/wiki/ARPANET) project, which is integrated into Unix BSD OS as the first implementation of `TCP/IP` protocols.

Nowadays, `TCP/IP` is still implemented in the operating system level. For Linux system, you can find the source code inside the kernel. The detailed implementation is outside the scope of this article. You can study it in this Github [link](https://github.com/torvalds/linux/tree/master/net/ipv4). 

### Socket

As I mentioned in the above sections, HTTP server is running in the application level. How it can work with `TCP/IP` stack which lives in the kernel? The answer is `socket`. 

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
	n, err := syscall.Read(ns.fd, p) // read from socket file descriptor
	if err != nil {
		n = 0
	}
	return n, err
}

func (ns netSocket) Write(p []byte) (int, error) {
	n, err := syscall.Write(ns.fd, p) // write to socket file descriptor
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

**netSocket** data model is created to represent the socket, which contains only one field **fd** means file descriptor. And all the socket related APIs: **Read**, **Write**,  **Accept** and **Close**, are defined. The usage of socket API is not in this article's scope, you can easily find a lot of great documents about it online. 

The logic of **netSocket** is not complicated, because it delegates the job to the kernel by `system call`. A system call is a programmatic way a program requests a service from the kernel, in detail you can refer to this [article](https://opensource.com/article/19/10/strace). In Golang, all the system calls are wrapped inside the `syscall` standard package.  

One thing need to mention is different platform have different `syscall` usages, so the demo code shown in this article can only be compiled and build on Linux system. 

Now we setup the TCP server and wait for connection request from client side. Next, let's see how to read or write HTTP request and response through socket. 
### HTTP

The main workflow is as follows: 

```golang
import (
	"flag"
	"http-server-scratch/simplenet"
	"log"
)

func main() {
	ipFlag := flag.String("ip_addr", "127.0.0.1", "The IP address to use")
	portFlag := flag.Int("port", 8080, "The port to use.")
	flag.Parse()

	ip := simplenet.ParseIP(*ipFlag)
	port := *portFlag
	socket, err := simplenet.NewNetSocket(ip, port)
	defer socket.Close()
	if err != nil {
		panic(err)
	}

	log.Print("===============")
	log.Print("Server Started!")
	log.Print("===============")
	log.Print()
	log.Printf("addr: http://%s:%d", ip, port)

	for {
		// Block until incoming connection
		rw, e := socket.Accept()
		log.Print()
		log.Print()
		log.Printf("Incoming connection")
		if e != nil {
			panic(e)
		}

		// Read request
		log.Print("Reading request")
		req, err := simplenet.ParseRequest(rw)
		log.Print("request: ", req)
		if err != nil {
			panic(err)
		}

		// Write response
		log.Print("Writing response")
		simplenet.WriteString(rw, "HTTP/1.1 200 OK\r\n"+
			"Content-Type: text/html; charset=utf-8\r\n"+
			"Content-Length: 20\r\n"+
			"\r\n"+
			"<h1>hello world</h1>")
		if err != nil {
			log.Print(err.Error())
			continue
		}
	}
}
```

As you can see, the HTTP request parsing logic is defined in the **ParseRequest** method in **simplenet** package. 

```golang
package simplenet

import (
	"bufio"
	"errors"
	"http-server-scratch/simplenet/simpleTextProto"
	"io"
	"strconv"
	"strings"
)

type request struct {
	method string // GET, POST, etc.
	header simpleTextProto.MIMEHeader
	body   []byte
	uri    string // The raw URI from the request
	proto  string // "HTTP/1.1"
}

func ParseRequest(c *netSocket) (*request, error) {
	b := bufio.NewReader(*c)
	tp := simpleTextProto.NewReader(b) // need replace
	req := new(request)

	// Parse request line: parse "GET /index.html HTTP/1.0"
	var s string
	s, _ = tp.ReadLine() // need replace
	sp := strings.Split(s, " ")
	req.method, req.uri, req.proto = sp[0], sp[1], sp[2]

	// Parse request headers
	mimeHeader, _ := tp.ReadMIMEHeader() // need replace
	req.header = mimeHeader

	// Parse request body
	if req.method == "GET" || req.method == "HEAD" {
		return req, nil
	}
	if len(req.header["Content-Length"]) == 0 {
		return nil, errors.New("no content length")
	}
	length, err := strconv.Atoi(req.header["Content-Length"][0])
	if err != nil {
		return nil, err
	}
	body := make([]byte, length)
	if _, err = io.ReadFull(b, body); err != nil {
		return nil, err
	}
	req.body = body
	return req, nil
}
```

The HTTP request message can be divided into three parts `request line`, `request headers` and `request body` as follows: 

<img src="/images/http_message_format.png" title="network" width="600px" height="400px">


The logic inside **ParseRequest** handles these 3 parts step by step. You can refer to the comments in the demo code. 

One thing need to emphasis is that **ParseRequest** method doesn't depends on `net` package. Because I want to show how HTTP server works in the bottom level, so I copy the request parsing logics from `net` package into my `simplenet` package. The parsing for request header part is kind of complex, but it doesn't influence your understanding about the main concept of HTTP server. If you want to know the details, you can refer to the `simplenet/simpleTextProto` package. The important thing to understand is HTTP server reads the request message with **Read** method of **netSocket** . And the **Read** method makes socket read system call to get network data from TCP stack: 

```golang
syscall.Read(ns.fd, p)
```

On the other side,  HTTP response is sent back by calling `WriteString` method of `simplenet` package


```golang
func WriteString(c *netSocket, s string) (n int, err error) {
	return c.Write([]byte(s))
}
```

`WriteString` simply calls **Write** method of **netsocket**, which makes socket write system call to send data over Interent with TCP stack:

```golang
syscall.Write(ns.fd, p)
```
That's all for the code part. Next let's try to run this simple HTTP server we build from scratch.

### Demo

Build (need Linux platform) and run this HTTP server with default options setting and send request to it with `curl`. The result goes as follows: 

<img src="/images/golang_http_server_demo.png" title="network" width="600px" height="400px">

the server works as expected. 







