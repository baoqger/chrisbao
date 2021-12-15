---
title: "How  HTTP1.1 protocol is implemented in Golang net/http package: part two -  write HTTP message to socket"
date: 2021-12-15 14:01:03
tags:
---

### Background

In the [previous](https://baoqger.github.io/2021/12/01/understand-http1-1-client-golang/) article, I introduced the main workflow of an HTTP request implemented inside Golang `net/http` package. As the second article of this series, I'll focus on how to pass the HTTP message to TCP/IP stack, and then it can be transported over the network. 

### Architecture diagram

When the client application sends an HTTP request, it determines what is next step based on whether there is an available persistent connection in the cached connection pool. If no, then a new TCP connection will be established. If yes, then a persistent connection will be selected. 

The details of the connection pool is not in this article's scope. I'll discuss it in the next article. For now you can regard it as a block box. 

The overall diagram of this article goes as follows, we can review each piece of it in the below sections

<img src="/images/golang-http1-1-flow-write-socket.png" title="write to socket" width="800px" height="600px">

### persistConn

The key structure in this part is `persistConn`: 
```golang
type persistConn struct {
	alt RoundTripper
	t         *Transport
	cacheKey  connectMethodKey
	conn      net.Conn            // underlying TCP connection
	tlsState  *tls.ConnectionState
	br        *bufio.Reader       
	bw        *bufio.Writer       // buffer io for writing data
	nwrite    int64               
	reqch     chan requestAndChan 
	writech   chan writeRequest   // channel for writing request
	closech   chan struct{}      
	isProxy   bool
	sawEOF    bool  
	readLimit int64 
	writeErrCh chan error
	writeLoopDone chan struct{} 
	idleAt    time.Time   
	idleTimer *time.Timer 
	mu                   sync.Mutex 
	numExpectedResponses int
	closed               error 
	canceledErr          error 
	broken               bool 
	reused               bool  
	mutateHeaderFunc func(Header)
}
```
There are many fields defined in `persistConn`, but we can focus on these three: 
- `conn`
- `bw`
- `writech`

