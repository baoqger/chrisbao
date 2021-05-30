---
title: "Circuit breaker and Hystrix: part two - timeout"
date: 2021-05-30 16:41:20
tags:
---

### Background

In the second article of this series, I will review the source code of `hystrix-go` project to understand how to design a `circuit breaker` and how to implement it with Golang. 

If you're not familiar with `circuit breaker` pattern or `hystrix-go` project, please check my previous [article](https://baoqger.github.io/2021/05/21/hystric-circuit-breaker-part1/) about it.  


### Three service degradation strategies



