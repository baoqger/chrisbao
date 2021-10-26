---
title: Understand how HTTP/1.1 persistent connection works based on Golang
date: 2021-10-25 11:11:28
tags: HTTP/1.1, persistent connection, keep-alive, TCP, netstat, tcpdump, Golang, connection pool
---

### Background

Initially, `HTTP` was a single request-and-response model. A `HTTP` client opens the `TCP` connection, requests a resource, gets the response, and the connection is closed. And establishing and terminating each `TCP` connection is a resource-consuming operation (in detail, you can refer to my previous [article](https://baoqger.github.io/2019/07/14/why-tcp-four-way-handshake/)). As the web application became more and more complex, displaying a single page may require several HTTP requests, too many TCP connection operations will have a bad impact on the performance. 

<img src="/images/short-lived-connection.png" title="tcp termination" width="400px" height="300px">

So `persistent-connection` (which is also called `keep-alive`) model is created in `HTTP/1.1` protocol. In this model TCP connections keep opened between several successive requests, in this way the time needed to open new connections will be reduced. 

<img src="/images/persistent-http.png" title="tcp termination" width="400px" height="300px">

In this article, I will show you how `persistent connection` works based on a Golang application. We will do some experiment based on the demo app, and verify the TCP connection behavior with some popular network packet analysis tools. In short, After reading this article, you will learn:
- Golang `http.Client` usage (and a little bit source code analysis)
- network analysis with `netstat` and `tcpdump`


Outline:
1. background: why we need persistent connection.

2. theory: how persistent connection works. In Both HTTP level and TCP level (keep alive for HTTP and TCP is not the same thing add a link to another post)

3. demo 1: sequence case
   1. non persistent connection: demonstrated by netstat command show tcpdump command result 
   2. persistent connection
   3. no source code review, link to next post
4. demo 2: concurrent case
   1. default connection pool and show result with netstat result
   2. a little bit source code to explain the Transport and connection pool
   3. tunning connection pool and show result with netstat result
   4. Wait_time state and port exhaustion
   5. HTTP2
