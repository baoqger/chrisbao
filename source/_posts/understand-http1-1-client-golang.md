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

First, the public `Get` method calls Get method of `DefaultClient`, which is a global variable of type `Client`,  

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
Note that by default the HTTP protocol version is set to 1.1. If you want to send HTTP2 request, then you need other solutions, and I'll write about it in other articles.  

Next, `Do` method is called which delegates the work to the private `do` method.  

```golang
func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}
```

`do` method handles the [`HTTP redirect`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections) behavior, which is very interesting. But since the code block is too long, I'll not show its function body here. You can refer to the source code of it [here](https://cs.opensource.google/go/go/+/refs/tags/go1.17.3:src/net/http/client.go;drc=refs%2Ftags%2Fgo1.17.3;l=598).



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

We already talked about the first parameter above. Let's take a look at the second parameter `c.transport()` as follows: 

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
I list the full content of `Transport` struct here, although it contains many fields, and many of them will not be discussed in this article.

As we just mentioned, `Transport` is type of `RoundTripper` interface, it must implement the method `RoundTrip`, right? 

You can find the `RoundTrip` method implementation of `Transport` struct type in **roundtrip.go** file as follows:

```golang
// RoundTrip method in roundtrip.go
func (t *Transport) RoundTrip(req *Request) (*Response, error) {
	return t.roundTrip(req)
}
```

Note at the beginning, I thought this method should be included inside `transport.go` file, but in fact it is defined inside another file.  

Let's back to the `send` method which takes `c.Transport` as the second argument:  

```golang
// send method in client.go 
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	req := ireq // req is either the original request, or a modified fork

	if rt == nil {
		req.closeBody()
		return nil, alwaysFalse, errors.New("http: no Client.Transport or DefaultTransport")
	}

	if req.URL == nil {
		req.closeBody()
		return nil, alwaysFalse, errors.New("http: nil Request.URL")
	}

	if req.RequestURI != "" {
		req.closeBody()
		return nil, alwaysFalse, errors.New("http: Request.RequestURI can't be set in client requests")
	}

	// forkReq forks req into a shallow clone of ireq the first
	// time it's called.
	forkReq := func() {
		if ireq == req {
			req = new(Request)
			*req = *ireq // shallow clone
		}
	}

	// Most the callers of send (Get, Post, et al) don't need
	// Headers, leaving it uninitialized. We guarantee to the
	// Transport that this has been initialized, though.
	if req.Header == nil {
		forkReq()
		req.Header = make(Header)
	}

	if u := req.URL.User; u != nil && req.Header.Get("Authorization") == "" {
		username := u.Username()
		password, _ := u.Password()
		forkReq()
		req.Header = cloneOrMakeHeader(ireq.Header)
		req.Header.Set("Authorization", "Basic "+basicAuth(username, password))
	}

	if !deadline.IsZero() {
		forkReq()
	}
	stopTimer, didTimeout := setRequestCancel(req, rt, deadline)

	resp, err = rt.RoundTrip(req)
	if err != nil {
		stopTimer()
		if resp != nil {
			log.Printf("RoundTripper returned a response & error; ignoring response")
		}
		if tlsErr, ok := err.(tls.RecordHeaderError); ok {
			// If we get a bad TLS record header, check to see if the
			// response looks like HTTP and give a more helpful error.
			// See golang.org/issue/11111.
			if string(tlsErr.RecordHeader[:]) == "HTTP/" {
				err = errors.New("http: server gave HTTP response to HTTPS client")
			}
		}
		return nil, didTimeout, err
	}
	if resp == nil {
		return nil, didTimeout, fmt.Errorf("http: RoundTripper implementation (%T) returned a nil *Response with a nil error", rt)
	}
	if resp.Body == nil {
		// The documentation on the Body field says “The http Client and Transport
		// guarantee that Body is always non-nil, even on responses without a body
		// or responses with a zero-length body.” Unfortunately, we didn't document
		// that same constraint for arbitrary RoundTripper implementations, and
		// RoundTripper implementations in the wild (mostly in tests) assume that
		// they can use a nil Body to mean an empty one (similar to Request.Body).
		// (See https://golang.org/issue/38095.)
		//
		// If the ContentLength allows the Body to be empty, fill in an empty one
		// here to ensure that it is non-nil.
		if resp.ContentLength > 0 && req.Method != "HEAD" {
			return nil, didTimeout, fmt.Errorf("http: RoundTripper implementation (%T) returned a *Response with content length %d but a nil Body", rt, resp.ContentLength)
		}
		resp.Body = ioutil.NopCloser(strings.NewReader(""))
	}
	if !deadline.IsZero() {
		resp.Body = &cancelTimerBody{
			stop:          stopTimer,
			rc:            resp.Body,
			reqDidTimeout: didTimeout,
		}
	}
	return resp, nil, nil
}
```

At **line 50** of `send` method above: 

```golang
resp, err = rt.RoundTrip(req)
```

`RoundTrip` method is called to send the request. Based on the comments in the source code, you can understand it in the following way:

- RoundTripper is an interface representing the ability to execute a single HTTP transaction, obtaining the Response for a given Request.

Next, let's go to `roundTrip` method of `Transport`: 

```golang
// roundTrip method in transport.go, which is called by RoundTrip method internally 

// roundTrip implements a RoundTripper over HTTP.
func (t *Transport) roundTrip(req *Request) (*Response, error) {
	t.nextProtoOnce.Do(t.onceSetNextProtoDefaults)
	ctx := req.Context()
	trace := httptrace.ContextClientTrace(ctx)

	if req.URL == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.URL")
	}
	if req.Header == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.Header")
	}
	scheme := req.URL.Scheme
	isHTTP := scheme == "http" || scheme == "https"
	if isHTTP {
		for k, vv := range req.Header {
			if !httpguts.ValidHeaderFieldName(k) {
				req.closeBody()
				return nil, fmt.Errorf("net/http: invalid header field name %q", k)
			}
			for _, v := range vv {
				if !httpguts.ValidHeaderFieldValue(v) {
					req.closeBody()
					return nil, fmt.Errorf("net/http: invalid header field value %q for key %v", v, k)
				}
			}
		}
	}

	origReq := req
	cancelKey := cancelKey{origReq}
	req = setupRewindBody(req)

	if altRT := t.alternateRoundTripper(req); altRT != nil {
		if resp, err := altRT.RoundTrip(req); err != ErrSkipAltProtocol {
			return resp, err
		}
		var err error
		req, err = rewindBody(req)
		if err != nil {
			return nil, err
		}
	}
	if !isHTTP {
		req.closeBody()
		return nil, badStringError("unsupported protocol scheme", scheme)
	}
	if req.Method != "" && !validMethod(req.Method) {
		req.closeBody()
		return nil, fmt.Errorf("net/http: invalid method %q", req.Method)
	}
	if req.URL.Host == "" {
		req.closeBody()
		return nil, errors.New("http: no Host in request URL")
	}

	for {
		select {
		case <-ctx.Done():
			req.closeBody()
			return nil, ctx.Err()
		default:
		}

		// treq gets modified by roundTrip, so we need to recreate for each retry.
		treq := &transportRequest{Request: req, trace: trace, cancelKey: cancelKey}
		cm, err := t.connectMethodForRequest(treq)
		if err != nil {
			req.closeBody()
			return nil, err
		}

		// Get the cached or newly-created connection to either the
		// host (for http or https), the http proxy, or the http proxy
		// pre-CONNECTed to https server. In any case, we'll be ready
		// to send it requests.
		pconn, err := t.getConn(treq, cm)
		if err != nil {
			t.setReqCanceler(cancelKey, nil)
			req.closeBody()
			return nil, err
		}

		var resp *Response
		if pconn.alt != nil {
			// HTTP/2 path.
			t.setReqCanceler(cancelKey, nil) // not cancelable with CancelRequest
			resp, err = pconn.alt.RoundTrip(req)
		} else {
			resp, err = pconn.roundTrip(treq)
		}
		if err == nil {
			resp.Request = origReq
			return resp, nil
		}

		// Failed. Clean up and determine whether to retry.
		if http2isNoCachedConnError(err) {
			if t.removeIdleConn(pconn) {
				t.decConnsPerHost(pconn.cacheKey)
			}
		} else if !pconn.shouldRetryRequest(req, err) {
			// Issue 16465: return underlying net.Conn.Read error from peek,
			// as we've historically done.
			if e, ok := err.(transportReadFromServerError); ok {
				err = e.err
			}
			return nil, err
		}
		testHookRoundTripRetried()

		// Rewind the body if we're able to.
		req, err = rewindBody(req)
		if err != nil {
			return nil, err
		}
	}
}
```

There are three key points:
- at **line 70**, a new variable of type `transportRequest`, which embeds `Request`, is created.  
- at **line 81**, `getConn` method is called, which implements the cached `connection pool` to support the `persistent connection` mode. Of course, if no cached connection is available, a new connection will be created and added to the connection pool. I will explain this behavior in detail next section. 
- from **line 89** to **line 95**, `pconn.roundTrip` is called. The name of variable `pconn` is self-explaining which means it is type of `persistConn`. 

Next, let's understand how `getConn` works. The logic can be summarized as the following diagram:

<img src="/images/golang-http1-1-getconn.png" title="getconn" width="800px" height="600px">




