---
title: "How to design a load performance test CLI tool"
date: 2021-07-04 10:18:43
tags:
---

### Background

When you want to do load performance test to your HTTP backend service, a handy and powerful tool can make your job mush easier. For example, `ApacheBench` (short for [ab](https://en.wikipedia.org/wiki/ApacheBench)) is widely used in this field. But it is not today's topic, instead I want to introduce [Hey](https://github.com/rakyll/hey) which is written in `Golang` and supports the same functionality as `ab`.  

`Hey` usage goes as follows:

```golang
Usage: hey [options...] <url>

Options:
  -n  Number of requests to run. Default is 200.
  -c  Number of workers to run concurrently. Total number of requests cannot
      be smaller than the concurrency level. Default is 50.
  -q  Rate limit, in queries per second (QPS) per worker. Default is no rate limit.
  -z  Duration of application to send requests. When duration is reached,
      application stops and exits. If duration is specified, n is ignored.
      Examples: -z 10s -z 3m.
  ...
// other options are hidden
```

I didn't list all the options, but just show several ones related to this article's content. As you can see in the above list, `Hey` can support different practical features, such as **multiple workers** to run in the **concurrent** style and **rate limit** by **queries per second (QPS)**. It can also set the number of requests to send or the time duration of sending requests.

In this article, we can review the design and implementation of `Hey` to see how to make a load performance testing tool.


### Architecture Design
The design of `Hey` is not complex can be devided into the following three parts:
- Control logic: the main workflow like how to set up multiple concurrent workers, how to control QPS rate limiter and how to exit the process when duration is reached; 
- HTTP request configuration: the headers or parameters configuration needed to send  request;   
- Test report: print or save the result after the load testing finished. 


