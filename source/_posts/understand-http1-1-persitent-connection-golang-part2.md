---
title: understand-http1-1-persitent-connection-golang-part2
date: 2021-10-27 17:51:29
tags:
---

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