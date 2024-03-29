---
title: "Understand how HTTP/1.1 persistent connection works based on Golang: part one - sequential requests" 
date: 2021-10-25 11:11:28
tags: HTTP/1.1, persistent connection, keep-alive, TCP, netstat, tcpdump, Golang, connection pool
---

### Background

Initially, `HTTP` was a single request-and-response model. An `HTTP` client opens the `TCP` connection, requests a resource, gets the response, and the connection is closed. And establishing and terminating each `TCP` connection is a resource-consuming operation (in detail, you can refer to my previous [article](https://baoqger.github.io/2019/07/14/why-tcp-four-way-handshake/)). As the web application becomes more and more complex, displaying a single page may require several HTTP requests, too many TCP connection operations will have a bad impact on the performance. 

<img src="/images/short-lived-connection.png" title="tcp termination" width="400px" height="300px">

So `persistent-connection` (which is also called `keep-alive`) model is created in `HTTP/1.1` protocol. In this model, TCP connections keep open between several successive requests, and in this way, the time needed to open new connections will be reduced. 

<img src="/images/persistent-http.png" title="tcp termination" width="400px" height="300px">

In this article, I will show you how `persistent connection` works based on a Golang application. We will do some experiments based on the demo app, and verify the TCP connection behavior with some popular network packet analysis tools. In short, After reading this article, you will learn:
- Golang `http.Client` usage (and a little bit source code analysis)
- network analysis with `netstat` and `tcpdump`

You can find the demo Golang application in this Github [repo](https://github.com/baoqger/http-persistent-connection-golang).

### Sequential requests

Let's start from the simple case where the client keeps sending `sequential` requests to the server. The [code](https://github.com/baoqger/http-persistent-connection-golang/blob/master/sequence/non-persistent-connection/non-persistent-connection.go) goes as follows: 

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

func startHTTPserver() {

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(50) * time.Microsecond)
		fmt.Fprintf(w, "Hello world")
	})

	go func() {
		http.ListenAndServe(":8080", nil)
	}()

}

func startHTTPRequest() {
	counter := 0
	for i := 0; i < 10; i++ {
		_, err := http.Get("http://localhost:8080/")
		if err != nil {
			panic(fmt.Sprintf("Error: %v", err))
		}
		log.Printf("HTTP request #%v", counter)
		counter += 1
		time.Sleep(time.Duration(1) * time.Second)
	}
}

func main() {
	startHTTPserver()

	startHTTPRequest()
}
```
We start an HTTP server in a Goroutine, and keep sending ten sequential requests to it. Right? Let's run the application and check the numbers and status of TCP connections.

After running the above code, you can see the following output: 

<img src="/images/sequential-request.png" title="tcp termination" width="800px" height="600px">

When the application stops running, we can run the following `netstat` command: 

```shell
netstat -n  | grep 8080
```

The TCP connections are listed as follows:

<img src="/images/sequential-request-netstat.png" title="tcp termination" width="800px" height="600px">

Obviously, the 10 HTTP requests are not persistent since 10 TCP connections are opened. 

**Note**: the last column of `netstat` shows the state of TCP connection. The state of TCP connection termination process can be explained with the following image: 

<img src="/images/state-tcp-connection.png" title="state tcp termination" width="600px" height="400px">

I will not cover the details in this article. But we need to understand the meaning of `TIME-WAIT`. 

In the `four-way handshake` process, the client will send the `ACK` packet to terminate the connection, but the state of TCP can't immediately go to `CLOSED`. The client has to wait for some time and the state in this waiting process is called `TIME-WAIT`. The TCP connection needs this `TIME-WAIT` state for two main reasons. 
- The first is to provide enough time that the `ACK` is received by the other peer. 
- The second is to provide a buffer period between the end of current connection and any subsequent ones. If not for this period, it's possible that packets from different connections could be mixed. In detail, you can refer to this [book](http://www.tcpipguide.com/free/t_TCPConnectionTermination-3.htm).

In our demo application case, if you wait for a while after the program stops, and run the `netstat` command again then no TCP connection will be listed in the output since they're all closed. 

Another tool to verify the TCP connections is `tcpdump`, which can capture every network packet send to your machine. In our case, you can run the following `tcpdump` command: 

```shell
sudo tcpdump -i any -n host localhost
```

It will capture all the network packets send from or to the localhost (we're running the server in localhost, right?). `tcpdump` is a great tool to help you understand the network, you can refer to its [document](https://www.tcpdump.org/) for more help.

**Note**: in our demo code above, we send 10 HTTP requests in sequence, which will make the capture result from `tcpdump` too long. So I modified the for loop to only send 2 sequential requests, which is enough to verify the behavior of `persistent connection`. The result goes as follows: 

<img src="/images/tcpdump-non-persistent.png" title="tcpdump" width="1200px" height="1000px">

In `tcpdump` output, the `Flag [S]` represents `SYN` flag, which is used to establish the TCP connection. The above snapshot contains two `Flag [S]` packets. The first `Flag [S]` is triggered by the first HTTP call, and the following packets are HTTP request and response. Then you can see the second `Flag [S]` packet to open a new TCP connection, which means the second HTTP request is not `persistent connection` as we hope. 

Next step, let's see how to make HTTP work as a persistent connection in Golang. 

In fact,this is a well known issue in Golang ecosystem, you can find the information in the [official document](https://pkg.go.dev/net/http#Client):

* If the returned error is nil, the Response will contain a non-nil Body which the user is expected to close. If the Body is not both read to EOF and closed, the Client's underlying RoundTripper (typically Transport) may not be able to re-use a persistent TCP connection to the server for a subsequent "keep-alive" request.

The fix will be straightforward by just adding two more lines of code as follows: 

```golang
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"time"
)

func startHTTPServer() {

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(50) * time.Microsecond)
		fmt.Fprintf(w, "Hello world")
	})

	go func() {
		http.ListenAndServe(":8080", nil)
	}()

}

func startHTTPRequest() {
	counter := 0
	for i := 0; i < 10; i++ {
		resp, err := http.Get("http://localhost:8080/")
		if err != nil {
			panic(fmt.Sprintf("Error: %v", err))
		}
		io.Copy(ioutil.Discard, resp.Body) // read the response body
		resp.Body.Close() // close the response body
		log.Printf("HTTP request #%v", counter)
		counter += 1
		time.Sleep(time.Duration(1) * time.Second)
	}
}

func main() {
	startHTTPServer()

	startHTTPRequest()
}
```

let's verify by running `netstat` command, the result goes as follows: 

<img src="/images/netstat-persistent.png" title="tcpdump" width="800px" height="600px">

This time 10 sequential HTTP requests establish only one TCP connection. This behavior is just what we hope: `persistent connection`.  

We can double verify it by doing the same experiment as above: run two HTTP requests in sequence and capture packets with `tcpdump`: 

<img src="/images/tcpdump-persistent.png" title="tcpdump" width="1200px" height="1000px">

This time, only one `Flag [S]` packet is there! The two sequential HTTP request re-use the same underlying TCP connection. 

### Summary

In this article, we showed how HTTP `persistent connection` works in the case of sequential requests. In the next article, we can show you the case of concurrent requests. 
