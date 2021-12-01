---
title: "Understand how HTTP/1.1 persistent connection works based on Golang: part two - concurrent requests"
date: 2021-10-27 17:51:29
tags: HTTP/1.1, concurrent
---

### Background

In the [last post](https://baoqger.github.io/2021/10/25/understand-http1-1-persistent-connection-golang/), I show you how HTTP/1.1 persistent connection works in a simple demo app, which sends sequential requests.

We observe the underlying TCP connection behavior based on the network analysis tool: `netstat` and `tcpdump`.

In this article, I will modify the demo app and make it send concurrent requests. In this way, we can have more understanding about HTTP/1.1's persistent connection.  

### Concurrent requests

The [demo code](https://github.com/baoqger/http-persistent-connection-golang/blob/master/concurrent/non-persistent-connection/non-persistent-connection-concurrent.go) goes as follows:  

```golang
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"sync"
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

func startHTTPRequest(index int, wg *sync.WaitGroup) {
	counter := 0
	for i := 0; i < 10; i++ {
		resp, err := http.Get("http://localhost:8080/")
		if err != nil {
			panic(fmt.Sprintf("Error: %v", err))
		}
		io.Copy(ioutil.Discard, resp.Body) // fully read the response body
		resp.Body.Close()                  // close the response body
		log.Printf("HTTP request #%v in Goroutine #%v", counter, index)
		counter += 1
		time.Sleep(time.Duration(1) * time.Second)
	}
	wg.Done()
}

func main() {
	startHTTPserver()
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go startHTTPRequest(i, &wg)
	}
	wg.Wait()
}

```
We create 10 goroutines, and each goroutine sends 10 sequential requests concurrently. 

**Note**: In HTTP/1.1 protocol, concurrent requests will establish multiple TCP connections. That's the restriction of HTTP/1.1, the way to enhance it is using `HTTP/2` which can multiplex one TCP connection for multiple parallel HTTP connections. `HTTP/2` is not in the scope of this post. I will talk about it in another article. 

Note that in the above demo, we have fully read the response body and closed it, and based on the discussion in [last article](https://baoqger.github.io/2021/10/25/understand-http1-1-persistent-connection-golang/), the HTTP requests should work in the persistent connection model. 

Before we use the network tool to analyze the behavior, let's imagine how many TCP connections will be established. As there are 10 concurrent goroutines, 10 TCP connections should be established, and all the HTTP requests should re-use these 10 TCP connections, right? That's our expectation. 

Next, let's verify our expectation with `netstat` as follows: 

<img src="/images/netstat-concurrent-non-persistent.png" title="tcp termination" width="600px" height="400px">

It shows that the number of TCP connections is much more than 10. The persistent connection does not work as we expect. 

After reading the source code of `net/http` package, I find the following hints: 

The `Client` is defined inside [client.go](https://golang.org/src/net/http/client.go) which is the type for HTTP client, and `Transport` is one of the properties.  
```golang
type Client struct {
	Transport RoundTripper

	CheckRedirect func(req *Request, via []*Request) error

	Jar CookieJar

	Timeout time.Duration
}
```

`Transport` is defined in [transport.go](https://golang.org/src/net/http/transport.go) like this: 

```golang
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2
```

`Transport` is type of  `RoundTripper`, which is an interface representing the ability to execute a single HTTP transaction, obtaining the Response for a given Request. `RoundTripper` is a very important structure in `net/http` package, we'll review (and analyze) the source code in the next article. In this article, we'll not discuss the details. 

Note that there are two parameters of `Transport`: 
- **MaxIdleConns**: controls the maximum number of idle (keep-alive) connections across all hosts.
- **MaxIdleConnsPerHost**: controls the maximum idle (keep-alive) connections to keep per-host. If zero, DefaultMaxIdleConnsPerHost is used.

By default, MaxIdleConns is **100** and MaxIdleConnsPerHost is **2**.

In our demo case, ten goroutines send requests to the same host (which is localhost:8080). Although MaxIdleConns is 100, but **only 2 idle connections can be cached** for this host because MaxIdleConnsPerHost is 2. That's why you saw much more TCP connections are established. 

Based on this analysis, let's refactor the code as follows: 

```golang
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"sync"
	"time"
)

var (
	httpClient *http.Client
)

func init() {
	httpClient = &http.Client{
		Transport: &http.Transport{
			MaxIdleConnsPerHost: 10, // set connection pool size for each host
			MaxIdleConns:        100,
		},
	}
}

func startHTTPserver() {

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(50) * time.Microsecond)
		fmt.Fprintf(w, "Hello world")
	})

	go func() {
		http.ListenAndServe(":8080", nil)
	}()

}

func startHTTPRequest(index int, wg *sync.WaitGroup) {
	counter := 0
	for i := 0; i < 10; i++ {
		resp, err := httpClient.Get("http://localhost:8080/")
		if err != nil {
			panic(fmt.Sprintf("Error: %v", err))
		}
		io.Copy(ioutil.Discard, resp.Body) // fully read the response body
		resp.Body.Close()                  // close the response body
		log.Printf("HTTP request #%v in Goroutine #%v", counter, index)
		counter += 1
		time.Sleep(time.Duration(1) * time.Second)
	}
	wg.Done()
}

func main() {
	startHTTPserver()
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go startHTTPRequest(i, &wg)
	}
	wg.Wait()
}
```
This time we don't use the default httpClient, instead we create a customized client which sets MaxIdleConnsPerHost to be **10**. This means the size of the connection pool is changed to 10, which can cache 10 idle TCP connections for each host.

Verify the behavior with `netstat` again: 

<img src="/images/netstat-concurrent-persistent.png" title="tcp termination" width="600px" height="400px">

Now the result is what we expect. 

### Summary

In this article, we discussed how to make HTTP/1.1 persistent connection work in a concurrent case by tunning the parameters for the connection pool. In the next article, let's review the source code to study how to implement HTTP client.