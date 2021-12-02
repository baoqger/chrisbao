---
title: How  HTTP1.1 protocol is implemented in Golang net/http package
date: 2021-12-01 10:02:34
tags: HTTP protocol, Golang, TCP, socket
---

### Background

In this article I will write about one topic: how to implement the HTTP protocol. In fact, I keep planning to write about this topic for a long time. In my previous articles, I already wrote several articles about HTTP protocol:

-  [How to write a Golang HTTP server with Linux system calls](https://baoqger.github.io/2021/07/31/how-to-implement-simple-http-server-golang/)
-  [Understand how HTTP/1.1 persistent connection works based on Golang: part one - sequential requests](https://baoqger.github.io/2021/10/25/understand-http1-1-persistent-connection-golang/)
-  [Understand how HTTP/1.1 persistent connection works based on Golang: part two - concurrent requests](https://baoqger.github.io/2021/10/27/understand-http1-1-persitent-connection-golang-part2/)

I recommended you to read these articles above before this one. 

As you know, HTTP protocol is in the application layer which is the closest one to the end user in the protocol stack. 

<img src="/images/osi_model.png" title="network" width="400px" height="300px">

So relatively speaking, HTTP protocol is not as mysterious as other protocols in the lower layers of this stack. As software engineer we use HTTP everyday and take it for granted. Have you ever think about how to implement a fully functional HTTP protocol library? 

It turns out to be a very complex and big work in terms of software engineering. Frankly speaking, I can't work it out by myself in a short period of time. So in this article, we'll try to understand how to do it by investigating Golang `net/http` package as an example. We'll read a lot of source code and draw diagrams to help your understand the source code.

**Note** HTTP protocol itself has evolved a lot from `HTTP1.1` to `HTTP2` and `HTTP3`, not to mention `HTTPS`. In this article, we'll focus on the mechanism of `HTTP1.1`, but what you learned here can help you understand other new versions of HTTP protocol. 

**Note** HTTP protocol is on the basis of client-server model. This article will focus on the client side or the HTTP request part. For the HTTP server part, I'll write another article in next step. 

### Main workflow of http.Client

HTTP client's request starts from the application's call to `Get` method of `net/http` package, and ends by writing the HTTP message to the TCP socket. The whole workflow can be simplified to the following diagram:   

<img src="/images/golang-http1-1-client-flow.png" title="golang client flow" width="800px" height="600px">

First, the `Get` method calls Get method of `DefaultClient`, which is a global variable of type `Client`,  

```golang
// Get method
func Get(url string) (resp *Response, err error) {
	return DefaultClient.Get(url)
}

// DefaultClient is a global variable in net/http package
var DefaultClient = &Client{}

// struct type Client
type Client struct {
	Transport RoundTripper

	CheckRedirect func(req *Request, via []*Request) error

	Jar CookieJar

	Timeout time.Duration
}
```
Then, `NewRequest` method is used to construct a new request of type `Request`: 

```golang
func (c *Client) Get(url string) (resp *Response, err error) {
	req, err := NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	return c.Do(req)
}


func NewRequest(method, url string, body io.Reader) (*Request, error) {
	return NewRequestWithContext(context.Background(), method, url, body)
}
```

I'll not show the function body of `NewRequestWithContext`, since it's very long. But only paste the block of code for actually building the `Request` object as follows:

```golang
req := &Request{
    // omit some code 
    Proto:      "HTTP/1.1", // the default HTTP protocol version is set to 1.1
    ProtoMajor: 1,
    ProtoMinor: 1,
    // omit some code
}
```
Next, `Do` method is called which delegates the work to the private `do` method.  

```golang
func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}
```

`do` method handles the [`HTTP redirect`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections) behavior, which is very interesting. But since the code block is too long, I'll not show its function body here. You can refer to the source code of it [here](https://cs.opensource.google/go/go/+/refs/tags/go1.17.3:src/net/http/client.go;drc=refs%2Ftags%2Fgo1.17.3;l=598).

Note that by default the HTTP protocol version is set to 1.1. If you want to send HTTP2 request, then you need other solutions, and I'll write about it in other articles.  

Next, `send` method of Client is called which goes as follows: 

```golang
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	if c.Jar != nil {
		for _, cookie := range c.Jar.Cookies(req.URL) {
			req.AddCookie(cookie)
		}
	}
    // call send method here
	resp, didTimeout, err = send(req, c.transport(), deadline)
	if err != nil {
		return nil, didTimeout, err
	}
	if c.Jar != nil {
		if rc := resp.Cookies(); len(rc) > 0 {
			c.Jar.SetCookies(req.URL, rc)
		}
	}
	return resp, nil, nil
}
```

It handles cookies for request, then call the private method `send` with there parameters.

We already talked about the first parameter above. Let's take a look at the second parameter as follows: 

```golang
func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}
```

`Transport` is extremely important for HTTP client workflow. Let's examine how it works bit by bit.  First of all, it's type of `RoundTripper` interface. 

```golang
// this interface is defined inside client.go file 

type RoundTripper interface {
	// RoundTrip executes a single HTTP transaction, returning
	// a Response for the provided Request.
	RoundTrip(*Request) (*Response, error)
}
```

`RoundTripper` interface only defines one method `RoundTrip`, all right. 

If you don't have any special settings, the `DefaultTransport` will be used for `c.Transport` above. 

The `DefaultTransport` is going as follows: 

```golang
// defined in transport.go
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```
Note that its actual type  is `Transport` as below:

```golang
type Transport struct {
	idleMu       sync.Mutex
	closeIdle    bool                                // user has requested to close all idle conns
	idleConn     map[connectMethodKey][]*persistConn // most recently used at end
	idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
	idleLRU      connLRU
	reqMu       sync.Mutex
	reqCanceler map[cancelKey]func(error)
	altMu    sync.Mutex   // guards changing altProto only
	altProto atomic.Value // of nil or map[string]RoundTripper, key is URI scheme
	connsPerHostMu   sync.Mutex
	connsPerHost     map[connectMethodKey]int
	connsPerHostWait map[connectMethodKey]wantConnQueue // waiting getConns
	Proxy func(*Request) (*url.URL, error)
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
	Dial func(network, addr string) (net.Conn, error)
	DialTLSContext func(ctx context.Context, network, addr string) (net.Conn, error)
	DialTLS func(network, addr string) (net.Conn, error)
	TLSClientConfig *tls.Config
	TLSHandshakeTimeout time.Duration
	DisableKeepAlives bool
	DisableCompression bool
	MaxIdleConns int
	MaxIdleConnsPerHost int
	MaxConnsPerHost int
	IdleConnTimeout time.Duration
	ResponseHeaderTimeout time.Duration
	ExpectContinueTimeout time.Duration
	TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper
	ProxyConnectHeader Header
	MaxResponseHeaderBytes int64
	WriteBufferSize int
	ReadBufferSize int
	nextProtoOnce      sync.Once
	h2transport        h2Transport // non-nil if http2 wired up
	tlsNextProtoWasNil bool        // whether TLSNextProto was nil when the Once fired
	ForceAttemptHTTP2 bool
}
```
