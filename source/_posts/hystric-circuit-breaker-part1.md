---
title: "Circuit breaker and Hystrix: part one - introduction"
date: 2021-05-21 15:25:19
tags:
---

In this series of articles, I want to talk about `circuit breaker` pattern based on an popular open source project `hystrix` (in fact, I will take a look at the golang version [hystrix-go](https://github.com/afex/hystrix-go), instead of the [original version](https://github.com/Netflix/Hystrix) which is written in Java).

As the first article of this series, I will give a general introduction to `circuit breaker`, let you know what it is and why it is important. Moreover, let's review the background about the project `hystrix-go` and `hystrix`, and understand the basic usage with a small demo example. 

### Circuit breaker

Software in distributed architectures generally have many dependencies, and the failure at some point for each dependency(even the most reliable service) is inevitable. 

What happens if our failing service becomes unresponsive? All services that rely on it have risks to become unresponsive, too. This is called `catastrophic cascading failure.`

The basic idea behind the circuit breaker is very simple. A circuit breaker works by wrapping calls to a target service and keeps monitoring the failure rates. Once the failures reach a certain threshold, the circuit breaker will trip ，and all the further calls to the circuit return with a fault or error. 

The design philosophy behind the circuit breaker pattern is `fail fast`: when a service becomes unresponsive, other services relying on it should stop waiting for it and start dealing with the fact that the failing service may be unavailable. By preventing a single service's failure cascading through the entire system, the circuit breaker pattern contributes to the `stability` and `resilience` of the whole system.  

The circuit breaker pattern can be implemented as a finite-state machine shown below:

![circuit-breaker](/images/circuit-breaker.png)

There are three statuses: `open`, `closed` and `half-open`

- **closed**: Requests are passed to the target service. Keep monitoring the metrics like error rate, request numbers and timeout. When these metrics exceed a specific threshold(which is set by the developer), the breaker is tripped and transitions into `open` status.   
- **open**: Requests are not passed to the target service, instead the `fallback` logic(which is defined by developer as well) will be called to handle the failure. The breaker will stay `open` status for a period of time called `sleeping window`, after which the breaker can transition from `open` to `half-open`.  
- **half-open**: In this status, a limited number of requests are passed to the target service, which is aims at resetting the status. If the target service can response successfully then the break is `reset` back to `closed` status. Or else the breaker transitions back to `open` status. 

That's basic background about circuit breaker, you can find much more [information](https://martinfowler.com/bliki/CircuitBreaker.html) about it on line. 

Next, let's investigate the project `hystrix`. 

### hystrix

`hystrix` is a very popular open source project. You can find everything about it in this [link](https://github.com/Netflix/Hystrix/wiki). 

I want to quote several important points from the above link. Hystrix is designed to do the following:
- Give protection from and control over latency and failure from dependencies accessed (typically over the network) via third-party client libraries.
- Stop cascading failures in a complex distributed system.
- Fail fast and rapidly recover.
- Fallback and gracefully degrade when possible.
- Enable near real-time monitoring, alerting, and operational control.

You can see `hystrix` perfectly implements the idea of circuit breaker pattern we talked about in the last section, right? 

The `hystrix` project is developed with `Java`. In this sereis of articles I prefer to use a golang version `hystrix-go`, which is a simplified version but implements all the main designs and ideas about circuit breaker. 

For the usage of `hystrix-go`, you can find it in this [link](https://github.com/afex/hystrix-go), which is very straightforward to understand. And you can easily find many other articles online with demo examples to show more usage level stuff. Please go head to read.

In my articles, I want to go into the source code of `hystrix-go` and have an advanced investigation about how `circuit breaker` is implemented. Please follow up to read the next articles in this series. 

### Summary

In this article, I talked about the background of circuit breaker pattern and the basic information of the popular open-source project in this field `hystrix-go`. Next step, we will take an in-depth look at the source code of this project. 

