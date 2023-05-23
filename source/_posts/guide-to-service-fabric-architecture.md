---
title: "Build Microservices with Service Fabric: A Hands-on Approach"
date: 2023-03-19 18:18:32
tags: Service Fabric, stateless, stateful, actor model, scalability, reliability,  partition
---

### Background

In this article, I want to introduce a distributed systems platform: [`Service Fabric`](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-overview), that is used to build microservices. 

First things first, what is `Service Fabric`? In the software world, `Service Fabric` is in the scope of the `orchestrator`. So it is the competitor of [`Kubernetes`](https://kubernetes.io/), [`Docker Swarm`](https://docs.docker.com/engine/swarm/), etc. 

I know what you're thinking about, at the time of writing, `Kubernetes` already won the competition. Why am I still writing about `Service Fabric`? My motivation to write this post is as follows:

- Firstly, I once used `Service Fabric` to build microservices and learned something valuable about it. I want to summarize all my learnings and share them with you here! 
- Secondly, we can (simply) examine both `Service Fabric` and `Kubernetes` to understand why the cloud-native solution is better and what problems it can solve. 
- Finally, `Service Fabric` is still widely used in some enterprise applications, in detail, you can refer [here](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-application-scenarios). So the skills you learned about service fabric is valuable. 

### Hands-on Project

To learn the `Service Fabric` platform, we will examine an open-source project: [HealthMetrics](https://github.com/baoqger/service-fabric-dotnet-data-aggregation). This project is originally developed by Microsoft itself to demonstrate the power of service fabric and I added some enhancements to it, in detail please refer to this [GitHub repo](https://github.com/baoqger/service-fabric-dotnet-data-aggregation).

At the business requirement level, this application goes like this: each `patient` keeps reporting the heart rate data to the `doctor` via the wearable device(the `band` in this case). And the `doctor` aggregates the collected health data and reports to the `county` service. Each `county` service instance runs some statistics on doctors' data and sends the report to the `country` service. Finally, you can get something as follows: 

<img src="/images/health-metrics-ui.png" title="health metrics" width="800px" height="600px">

Before we explore the detailed implementation, you can pause here for a few minutes. How will you design the architecture if this application is assigned to you? How to make it both `reliable` and `scalable`?  

### Service Fabric Programming Model

`Service Fabric` provides multiple ways to write and manage your services: 

- Reliable Services: is the most popular choice in the context of service fabric, we'll examine much more about it later. 
- Containers: service fabric can also deploy services in containers. But if you choose to use containers, why not directly use `Kubernetes`? I'll not cover it in this post. 
- Guest executables: You can run any type of code or script like Node.js as guest executables in the service fabric. Note that the service fabric platform doesn't include the runtime for the guest executables, instead, the operating system hosting the service fabric must provide the corresponding runtime for the given language. It provides a flexible way to run legacy applications within microservices. If you have an interest in it, please refer to this [article](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-guest-executables-introduction), I will not examine the details in this post.

Next, let's examine what is `Reliable Services`? 

### Reliable Services

<img src="/images/healthmetrics-architecture.png" title="architecture" width="800px" height="600px">




提纲:

service fabric vs k8s, 简单带过

focus on development part, 不包含deployment

理论讲解: programming model: stateless, stateful, actor. 都要介绍下

结合项目介绍: 细致分析

服务通信: RPC, message passing, async of message broker(event-driven application), internal DNS

scalability and availability:  instance(actor service), partition(county service) and replicas


展开讨论: 

service fabric的问题, service mesh的优势