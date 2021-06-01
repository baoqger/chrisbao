---
title: "Circuit breaker and Hystrix: part two - timeout"
date: 2021-05-30 16:41:20
tags:
---

### Background

In the second article of this series, I will review the source code of `hystrix-go` project to understand how to design a `circuit breaker` and how to implement it with Golang. 

If you're not familiar with `circuit breaker` pattern or `hystrix-go` project, please check my previous [article](https://baoqger.github.io/2021/05/21/hystric-circuit-breaker-part1/) about it.  


### Three service degradation strategies

`Hystrix` provides three different service degradation strategies to avoid the `cascading failure` happening in the entire system: `timeout`, `maximum concurrent request numbers` and `request error rate`. 

- timeout: if the service call doesn't return response successfully within a predefined time duration, then the fallback logic will run. This strategy is the simplest one. 
- maximum concurrent request numbers: when the number of concurrent requests is beyond the threshold, then the fallback logic will handle the following request. 
- request error rate: `hystrix` will record the response status of each service call, after the error rate reaches the threshold, the breaker will be open, and the fallback logic will execute before the breaker status changes back to closed. `error rate` strategy is the most complex one. 



