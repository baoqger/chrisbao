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
- `conn`: type of `net.Conn` which defines TCP connection in Golang;
- `bw`: type of `*bufio.Writer` which implements `buffer io` functionality;
- `writech`: type of `channel` which is used to communicate and sync data among different Goroutines in Golang.

In next sections, let's investigate how `persistConn` is used to write HTTP message to socket. 

### New connection

First, let's see how to establish a new TCP connection and bind it to `persistConn` structure. The job is done inside **dialConn** method of **Transport**

```golang
// dialConn in transport.go file
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
	// construct a new persistConn
	pconn = &persistConn{
		t:             t,
		cacheKey:      cm.key(),
		reqch:         make(chan requestAndChan, 1),
		writech:       make(chan writeRequest, 1),
		closech:       make(chan struct{}),
		writeErrCh:    make(chan error, 1),
		writeLoopDone: make(chan struct{}),
	}
	trace := httptrace.ContextClientTrace(ctx)
	wrapErr := func(err error) error {
		if cm.proxyURL != nil {
			return &net.OpError{Op: "proxyconnect", Net: "tcp", Err: err}
		}
		return err
	}
	if cm.scheme() == "https" && t.hasCustomTLSDialer() {
		var err error
		// dial secure TCP connection, assign to field pconn.conn
		pconn.conn, err = t.customDialTLS(ctx, "tcp", cm.addr())
		if err != nil {
			return nil, wrapErr(err)
		}
		if tc, ok := pconn.conn.(*tls.Conn); ok {
			if trace != nil && trace.TLSHandshakeStart != nil {
				trace.TLSHandshakeStart()
			}
			if err := tc.Handshake(); err != nil {
				go pconn.conn.Close()
				if trace != nil && trace.TLSHandshakeDone != nil {
					trace.TLSHandshakeDone(tls.ConnectionState{}, err)
				}
				return nil, err
			}
			cs := tc.ConnectionState()
			if trace != nil && trace.TLSHandshakeDone != nil {
				trace.TLSHandshakeDone(cs, nil)
			}
			pconn.tlsState = &cs
		}
	} else {
		// dial TCP connection
		conn, err := t.dial(ctx, "tcp", cm.addr())
		if err != nil {
			return nil, wrapErr(err)
		}
		// assign to pconn.conn
		pconn.conn = conn
		if cm.scheme() == "https" {
			var firstTLSHost string
			if firstTLSHost, _, err = net.SplitHostPort(cm.addr()); err != nil {
				return nil, wrapErr(err)
			}
			if err = pconn.addTLS(firstTLSHost, trace); err != nil {
				return nil, wrapErr(err)
			}
		}
	}
	switch {
	case cm.proxyURL == nil:
	case cm.proxyURL.Scheme == "socks5":
		conn := pconn.conn
		d := socksNewDialer("tcp", conn.RemoteAddr().String())
		if u := cm.proxyURL.User; u != nil {
			auth := &socksUsernamePassword{
				Username: u.Username(),
			}
			auth.Password, _ = u.Password()
			d.AuthMethods = []socksAuthMethod{
				socksAuthMethodNotRequired,
				socksAuthMethodUsernamePassword,
			}
			d.Authenticate = auth.Authenticate
		}
		if _, err := d.DialWithConn(ctx, conn, "tcp", cm.targetAddr); err != nil {
			conn.Close()
			return nil, err
		}
	case cm.targetScheme == "http":
		pconn.isProxy = true
		if pa := cm.proxyAuth(); pa != "" {
			pconn.mutateHeaderFunc = func(h Header) {
				h.Set("Proxy-Authorization", pa)
			}
		}
	case cm.targetScheme == "https":
		conn := pconn.conn
		hdr := t.ProxyConnectHeader
		if hdr == nil {
			hdr = make(Header)
		}
		if pa := cm.proxyAuth(); pa != "" {
			hdr = hdr.Clone()
			hdr.Set("Proxy-Authorization", pa)
		}
		connectReq := &Request{
			Method: "CONNECT",
			URL:    &url.URL{Opaque: cm.targetAddr},
			Host:   cm.targetAddr,
			Header: hdr,
		}

		connectCtx := ctx
		if ctx.Done() == nil {
			newCtx, cancel := context.WithTimeout(ctx, 1*time.Minute)
			defer cancel()
			connectCtx = newCtx
		}

		didReadResponse := make(chan struct{}) 
		var (
			resp *Response
			err  error 
		)

		go func() {
			defer close(didReadResponse)
			err = connectReq.Write(conn)
			if err != nil {
				return
			}
			br := bufio.NewReader(conn)
			resp, err = ReadResponse(br, connectReq)
		}()
		select {
		case <-connectCtx.Done():
			conn.Close()
			<-didReadResponse
			return nil, connectCtx.Err()
		case <-didReadResponse:

		}
		if err != nil {
			conn.Close()
			return nil, err
		}
		if resp.StatusCode != 200 {
			f := strings.SplitN(resp.Status, " ", 2)
			conn.Close()
			if len(f) < 2 {
				return nil, errors.New("unknown status code")
			}
			return nil, errors.New(f[1])
		}
	}

	if cm.proxyURL != nil && cm.targetScheme == "https" {
		if err := pconn.addTLS(cm.tlsHost(), trace); err != nil {
			return nil, err
		}
	}

	if s := pconn.tlsState; s != nil && s.NegotiatedProtocolIsMutual && s.NegotiatedProtocol != "" {
		if next, ok := t.TLSNextProto[s.NegotiatedProtocol]; ok {
			alt := next(cm.targetAddr, pconn.conn.(*tls.Conn))
			if e, ok := alt.(http2erringRoundTripper); ok {
				return nil, e.err
			}
			return &persistConn{t: t, cacheKey: pconn.cacheKey, alt: alt}, nil
		}
	}
	pconn.br = bufio.NewReaderSize(pconn, t.readBufferSize())
	// buffer io wrapper for writing request
	pconn.bw = bufio.NewWriterSize(persistConnWriter{pconn}, t.writeBufferSize())
	// read loop
	go pconn.readLoop()
	// write loop
	go pconn.writeLoop()
	return pconn, nil
}
```
At **line 4**, it creates a new `persistConn` object, which is also the return value for this method. 

At **line 22** and **line 46**, it calls `dial` method to establish a new TCP connection (note line 22 handles `TLS` case). In Golang a TCP connection is represented as `net.Conn` type. And then the underlying TCP connection is bound to the `conn` field of `persistConn`. 

Now that we have the TCP connection, how can we use it? We'll skip the many lines of code and go to the end to this function. 

At **line 166**,  it creates `bufio.Writer` based on `persistConn`. `Buffer IO` is an interesting topic, in detail you can refer to my previous [article](https://baoqger.github.io/2021/04/04/golang-bytes-buffer/). In one word, it can optimize the performance by reducing the number of system calls. For example in the current case, it can avoid too many `socket` system calls. 

At **line 171**, it creates a Goroutine and execute `writeLoop` method. Let's take a look at it. 

### writeLoop

```golang
// writeLoop method in transport.go file
func (pc *persistConn) writeLoop() {
	defer close(pc.writeLoopDone)
	for {
		select {
		// receive request from writech channel
		case wr := <-pc.writech:
			startBytesWritten := pc.nwrite
			// call write method
			err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
			if bre, ok := err.(requestBodyReadError); ok {
				err = bre.error
				wr.req.setError(err)
			}
			if err == nil {
				err = pc.bw.Flush()
			}
			if err != nil {
				wr.req.Request.closeBody()
				if pc.nwrite == startBytesWritten {
					err = nothingWrittenError{err}
				}
			}
			pc.writeErrCh <- err // to the body reader, which might recycle us
			wr.ch <- err         // to the roundTrip function
			if err != nil {
				pc.close(err)
				return
			}
		case <-pc.closech:
			return
		}
	}
}
```
As the function name **writeLoop** implies, there is a **for** loop, and it keeps receiving data from the **writech** channel. Everytime it receive a request from the channel, call the `write` method at **line 10**. Then let's review what message it actually writes:

```golang
// write method in request.go
func (r *Request) write(w io.Writer, usingProxy bool, extraHeaders Header, waitForContinue func() bool) (err error) {
	trace := httptrace.ContextClientTrace(r.Context())
	if trace != nil && trace.WroteRequest != nil {
		defer func() {
			trace.WroteRequest(httptrace.WroteRequestInfo{
				Err: err,
			})
		}()
	}
	host := cleanHost(r.Host)
	if host == "" {
		if r.URL == nil {
			return errMissingHost
		}
		host = cleanHost(r.URL.Host)
	}
	host = removeZone(host)
	ruri := r.URL.RequestURI()
	if usingProxy && r.URL.Scheme != "" && r.URL.Opaque == "" {
		ruri = r.URL.Scheme + "://" + host + ruri
	} else if r.Method == "CONNECT" && r.URL.Path == "" {
		ruri = host
		if r.URL.Opaque != "" {
			ruri = r.URL.Opaque
		}
	}
	if stringContainsCTLByte(ruri) {
		return errors.New("net/http: can't write control character in Request.URL")
	}
	var bw *bufio.Writer
	if _, ok := w.(io.ByteWriter); !ok {
		bw = bufio.NewWriter(w)
		w = bw
	}
	// write HTTP request line
	_, err = fmt.Fprintf(w, "%s %s HTTP/1.1\r\n", valueOrDefault(r.Method, "GET"), ruri)
	if err != nil {
		return err
	}
	// write HTTP request Host header 
	_, err = fmt.Fprintf(w, "Host: %s\r\n", host)
	if err != nil {
		return err
	}
	if trace != nil && trace.WroteHeaderField != nil {
		trace.WroteHeaderField("Host", []string{host})
	}

	userAgent := defaultUserAgent
	if r.Header.has("User-Agent") {
		userAgent = r.Header.Get("User-Agent")
	}
	if userAgent != "" {
		// write HTTP request User-Agent header 
		_, err = fmt.Fprintf(w, "User-Agent: %s\r\n", userAgent)
		if err != nil {
			return err
		}
		if trace != nil && trace.WroteHeaderField != nil {
			trace.WroteHeaderField("User-Agent", []string{userAgent})
		}
	}

	tw, err := newTransferWriter(r)
	if err != nil {
		return err
	}
	err = tw.writeHeader(w, trace)
	if err != nil {
		return err
	}

	err = r.Header.writeSubset(w, reqWriteExcludeHeader, trace)
	if err != nil {
		return err
	}

	if extraHeaders != nil {
		err = extraHeaders.write(w, trace)
		if err != nil {
			return err
		}
	}
	// write blank line after HTTP request headers
	_, err = io.WriteString(w, "\r\n")
	if err != nil {
		return err
	}

	if trace != nil && trace.WroteHeaders != nil {
		trace.WroteHeaders()
	}

	if waitForContinue != nil {
		if bw, ok := w.(*bufio.Writer); ok {
			err = bw.Flush()
			if err != nil {
				return err
			}
		}
		if trace != nil && trace.Wait100Continue != nil {
			trace.Wait100Continue()
		}
		if !waitForContinue() {
			r.closeBody()
			return nil
		}
	}

	if bw, ok := w.(*bufio.Writer); ok && tw.FlushHeaders {
		if err := bw.Flush(); err != nil {
			return err
		}
	}
	err = tw.writeBody(w)
	if err != nil {
		if tw.bodyReadError == err {
			err = requestBodyReadError{err}
		}
		return err
	}

	if bw != nil {
		return bw.Flush()
	}
	return nil
}
```
We will not go through every line of code in above function. But I bet you find many familiar information, for example, at line 37 it write **HTTP request line** as the first information in the HTTP message. Then it continues writing **HTTP headers** such as **Host** and **User-Agent**(at line 42 and line 56), and finally add the **blank line** after the headers (at line 86). An HTTP request message is built up bit by bit. All right.  

### Bufio and underlying writer

Next piece of this puzzle is how it's related to the underlying TCP connection. 

Note this method call in the write loop: 

```golang
// write method call in writeLoop
wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
```
The first parameter is `pc.bw` mentioned above. It's time to take a deep look at it. `pc.bw`, a **bufio.Write**, is created by calling the following method from `bufio` package: 

```golang
// pconn.bw is created by this method call
pconn.bw = bufio.NewWriterSize(persistConnWriter{pconn}, t.writeBufferSize())
```

Note that this **bufio.Writer** isn't based on `persistConn` directly, instead a simple wrapper over `persistConn` called `persistConnWriter` is used here. 

```golang
// persistConnWriter in transport.go file
type persistConnWriter struct {
	pc *persistConn
}
```

What we need to understand is **bufio.Writer wraps an io.Writer object, creating another Writer that also implements the interface but provides buffering functionality.** And **bufio.Writer's Flush method writes the buffered data to the underlying io.Writer.**

In this case, the underlying io.Writer is `persistConnWriter`. Its `Write` method will be used to write the buffered data: 

```golang
// persistConnWriter in transport.go file
func (w persistConnWriter) Write(p []byte) (n int, err error) {
	n, err = w.pc.conn.Write(p) // TCP socket Write system call is called here!
	w.pc.nwrite += int64(n)
	return
}
```

Internally it delegates the task to the TCP connection bond to `pconn.conn`! 

### roundTrip

As we mentioned above, `writeLoop` keeps receiving reqeusts from `writech` channel. So on the other hand, it means the requests should be sent to this channel somewhere. This is implemented inside the `roundTrip` method: 

```golang
// roundTrip in transport.go file
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	testHookEnterRoundTrip()
	if !pc.t.replaceReqCanceler(req.cancelKey, pc.cancelRequest) {
		pc.t.putOrCloseIdleConn(pc)
		return nil, errRequestCanceled
	}
	pc.mu.Lock()
	pc.numExpectedResponses++
	headerFn := pc.mutateHeaderFunc
	pc.mu.Unlock()

	if headerFn != nil {
		headerFn(req.extraHeaders())
	}

	requestedGzip := false
	if !pc.t.DisableCompression &&
		req.Header.Get("Accept-Encoding") == "" &&
		req.Header.Get("Range") == "" &&
		req.Method != "HEAD" {
		requestedGzip = true
		req.extraHeaders().Set("Accept-Encoding", "gzip")
	}

	var continueCh chan struct{}
	if req.ProtoAtLeast(1, 1) && req.Body != nil && req.expectsContinue() {
		continueCh = make(chan struct{}, 1)
	}

	if pc.t.DisableKeepAlives && !req.wantsClose() {
		req.extraHeaders().Set("Connection", "close")
	}

	gone := make(chan struct{})
	defer close(gone)

	defer func() {
		if err != nil {
			pc.t.setReqCanceler(req.cancelKey, nil)
		}
	}()

	const debugRoundTrip = false
	startBytesWritten := pc.nwrite
	writeErrCh := make(chan error, 1)
	// send requet to pc.writech channel 
	pc.writech <- writeRequest{req, writeErrCh, continueCh}

	resc := make(chan responseAndError)
	pc.reqch <- requestAndChan{
		req:        req.Request,
		cancelKey:  req.cancelKey,
		ch:         resc,
		addedGzip:  requestedGzip,
		continueCh: continueCh,
		callerGone: gone,
	}

	var respHeaderTimer <-chan time.Time
	cancelChan := req.Request.Cancel
	ctxDoneChan := req.Context().Done()
	for {
		testHookWaitResLoop()
		select {
		case err := <-writeErrCh:
			if debugRoundTrip {
				req.logf("writeErrCh resv: %T/%#v", err, err)
			}
			if err != nil {
				pc.close(fmt.Errorf("write error: %v", err))
				return nil, pc.mapRoundTripError(req, startBytesWritten, err)
			}
			if d := pc.t.ResponseHeaderTimeout; d > 0 {
				if debugRoundTrip {
					req.logf("starting timer for %v", d)
				}
				timer := time.NewTimer(d)
				defer timer.Stop() 
				respHeaderTimer = timer.C
			}
		case <-pc.closech:
			if debugRoundTrip {
				req.logf("closech recv: %T %#v", pc.closed, pc.closed)
			}
			return nil, pc.mapRoundTripError(req, startBytesWritten, pc.closed)
		case <-respHeaderTimer:
			if debugRoundTrip {
				req.logf("timeout waiting for response headers.")
			}
			pc.close(errTimeout)
			return nil, errTimeout
		case re := <-resc:
			if (re.res == nil) == (re.err == nil) {
				panic(fmt.Sprintf("internal error: exactly one of res or err should be set; nil=%v", re.res == nil))
			}
			if debugRoundTrip {
				req.logf("resc recv: %p, %T/%#v", re.res, re.err, re.err)
			}
			if re.err != nil {
				return nil, pc.mapRoundTripError(req, startBytesWritten, re.err)
			}
			return re.res, nil
		case <-cancelChan:
			pc.t.cancelRequest(req.cancelKey, errRequestCanceled)
			cancelChan = nil
		case <-ctxDoneChan:
			pc.t.cancelRequest(req.cancelKey, req.Context().Err())
			cancelChan = nil
			ctxDoneChan = nil
		}
	}
}
```

At line 48, you can find it clearly. 

In [last article](https://baoqger.github.io/2021/12/01/understand-http1-1-client-golang/), you can find that `pconn.roundTrip` is the end of the HTTP request workflow. Now we had put all parts together. Great. 

### Summary

In this article (as the second part of this series), we reviewed how the HTTP request message is written to TCP/IP stack via socket system call.  


