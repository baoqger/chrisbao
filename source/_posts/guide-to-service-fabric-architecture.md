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

As I mentioned above, `service fabric` is an `orchestrator` which provides infrastructure-level functionalities like `cluster management`, `service discovery`, and `scaling` as `Kubernetes` does. But `service fabric` goes further by also providing a `reliable services` model to guide the development on the application level, this's a special point compared to `Kubernetes`. Simply speaking, the application code can call the `service fabric` runtime APIs to query the system for building reliable applications. In the following sections, I'll show you what that means with some code blocks. 

`Reliable Services` can be classified into two types as follows: 

- Stateless service: when we mentioned stateless service in the context of service fabric, we are not saying that the service doesn't have any state to store, but it means that the service doesn't store the data inside the cluster of service fabric. Indeed, the stateless service can store states in the external database. The naming of stateless is relative to the cluster. 
- Stateful service: similarly, the stateful service keeps its state locally in the service fabric cluster, which means the data and service are in the same virtual or physical machine. And this is called `Reliable Collections`. Compared with external data storage, `Reliable Collections` implies low latency and high performance. In the following section, I will introduce more about `Reliable Collections`, which is a very interesting topic. Please hold on!

<img src="/images/reliable-services.png" title="reliable services" width="600px" height="400px">

Besides stateless and stateful services listed above, there is the third type of service provided by Service Fabric: `Actor Service`. 

- Actor Service: is a special type of stateful service, which applies the `actor model` theory. The [`actor model`](https://en.wikipedia.org/wiki/Actor_model) is a general computer science theory handling `concurrent computation`. It is a very big topic worthy of a separate article. I'll not cover the details in this post, instead, let's go through this model in the context of service fabric. 

The `actor model` is proposed many years ago in 1973 to simplify the process of writing concurrent programs with the following several advantages: 

- Encapsulation: in the actor model, each actor is a self-contained unit that encapsulates its own state and behavior. And each actor runs `only one thread`, so you don't have to worry about complex multi-threading programming issues. But how does it support high concurrency? Just allocate `more actor instances` as the load goes up.   
- Message passing: different from the traditional multi-threading programming techniques, actors do not `share memories`, instead, they communicate with async message-passing communication, which can reduce the complexity of concurrent programs. 
- Location transparency: as a self-contained computation unit, each actor can be located on different machines, which makes it the perfect solution for building distributed applications as service fabric does. 

In the future, I will write an article for actor model to examine more details about it. Next, let's take a look at the demo app: [HealthMetrics](https://github.com/baoqger/service-fabric-dotnet-data-aggregation) and analyze how it was build bsaed on the programming models we discussed above. 

<img src="/images/actor-model.png" title="actor model" width="400px" height="300px">  

### Architecture of HealthMetrics

<img src="/images/healthmetrics-architecture.png" title="architecture" width="800px" height="600px">

There are several services included in this `HealthMetrics` demo application. They are: 

- BandCreationService: this `stateless` service read the input CSV file and creates the individual `band actors` and `doctor actors`. For example, as the above snapshot shows, the application creates roughly 300 band actors and doctor actors.  
- BandActor: this `actor` service is the host for the band actors. Each BandActor represents an individual wearable device, which generates heart rate data and then sends it to the designated DoctorActor every 5 seconds. 
- DoctorActor: the doctor `actor` service aggregates all of the data it receives from each band actor and generates an overall view for that doctor and then pushes it into the CountyService every 5 seconds.
- CountyService: this `stateful` service aggregates the information provided by the doctor actor further and also pushes the data into the NationalService. 
- NationalService: this `stateful` service maintains the total aggerated data for the entire county and is used to serve data requested by WebService. 
- WebService: this `stateless` service just hosts a simple web API to query information from NationalService and render the data on the web UI. 

As you can see, this demo application consists of all three types of services provided by service fabric. 

In the next section, let's examine what kinds of techniques of service fabric are applied to build this highly `scalable` and `available` microservices application. 