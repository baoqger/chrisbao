---
title: "Build Microservices with Service Fabric: A Hands-on Approach"
date: 2023-03-20 18:18:32
tags: Service Fabric, stateless, stateful, actor model, scalability, reliability,  partition
---

### Background

In this article, I want to introduce a distributed systems platform: [`Service Fabric`](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-overview), that is used to build microservices. 

First things first, what is `Service Fabric`? In the software world, `Service Fabric` is in the scope of the `orchestrator`. So it is the competitor of [`Kubernetes`](https://kubernetes.io/), [`Docker Swarm`](https://docs.docker.com/engine/swarm/), etc. 

I know what you're thinking about, at the time of writing, `Kubernetes` already won the competition. Why am I still writing about `Service Fabric`? My motivation to write this post is as follows:

- Firstly, I once used `Service Fabric` to build microservices and learned something valuable about it. I want to summarize all my learnings and share them with you here! 
- Secondly, we can (simply) examine both `Service Fabric` and `Kubernetes` to understand why the cloud-native solution is better and what problems it can solve. 
- Finally, `Service Fabric` is still widely used in some enterprise applications, in detail, you can refer [here](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-application-scenarios). So the skills you learned about service fabric are valuable. 

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

Next, let's examine what are `Reliable Services`.

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

In the future, I will write an article on the actor model to examine more details about it. Next, let's take a look at the demo app: [HealthMetrics](https://github.com/baoqger/service-fabric-dotnet-data-aggregation) and analyze how it was built based on the programming models we discussed above. 

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

As you can see, this demo application consists of all three types of services provided by service fabric, which is a perfect case to learn `service fabric`, right? 

In the next sections, let's examine what kinds of techniques of service fabric are applied to build this highly `scalable` and `available` microservices application. 



### Naming Perspective

`Naming` plays an important role in the `distributed system`. They are used to share resources, to uniquely identify entities, to refer to locations, and so on. In the `distributed system`, each name should be resolved to the entity it refers to. To resolve names, it is necessary to implement a `naming system`.

In `Service Fabric`, `naming` is used to identify and locate `services` and `actors` within a cluster. And `Service Fabric` uses a `structured naming` system, where services and actors are identified using a `hierarchical` naming scheme. The naming system is based on the concept of a `namespace`. A namespace is a logical grouping of related `applications` and `services`. In the `Service Fabric` namespace, the first level is the `application`, which represents a logical grouping of services, and the second level is the `service`, which represents a specific unit of functionality that is part of the application. And the default namespace is `fabric:/`, so the service [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) in the `Service Fabric` cluster is in the following format: 

```shell
fabric:/applicationName/ServiceName
```
And in our `HealthMetrics` demo app, the URL will be something like `fabric:/HealthMetrics/HealthMetrics.BandActor` or `fabric:/HealthMetrics/HealthMetrics.CountyService`(the application name is `HealthMetrics` and the service name is `HealthMetrics.BandActor` or `HealthMetrics.CountyService`). 

In this demo app, we build a helper class `ServiceUriBuilder` to generate URI as follows: 

```csharp
public Uri ToUri()
{
    string applicationInstance = this.ApplicationInstance;

    if (String.IsNullOrEmpty(applicationInstance))
    {
        try
        {
            // the ApplicationName property here automatically prepends "fabric:/" for us
            applicationInstance = FabricRuntime.GetActivationContext().ApplicationName.Replace("fabric:/", String.Empty);
        }
        catch (InvalidOperationException)
        {
            // FabricRuntime is not available. 
            // This indicates that this is being called from somewhere outside the Service Fabric cluster.
        }
    }

    return new Uri("fabric:/" + applicationInstance + "/" + this.ServiceInstance);
}
```

For detail, please refer to this source code [file](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetrics.Common/ServiceUriBuilder.cs).

Since the behavior of the name resolver is quite similar to the internet domain name resolver, the `Service Fabric Name Service` can be regarded as an internal [`DNS`](https://en.wikipedia.org/wiki/Domain_Name_System) service. This type of internal `DNS` service is a common requirement for all distributed systems, including `Kubernetes`. In the modern `cloud-native` ecosystem, the popular choice is [`CoreDNS`](https://coredns.io/), which runs inside K8S. In the future, I will write an article about DNS. Please keep watching my blog's update.

Besides this default `DNS` solution mentioned above, you can use other `service registry` and `service discovery` solutions. For example, in my [previous article](https://organicprogrammer.com/2020/11/16/golang-service-discovery-consul/), I once examined how to do this based on [`Consul`](https://www.consul.io/). Imagine your large-scale application consists of hundreds of microservices, with the default DNS solutions you have to hardcode so many names in each service. That's just where `Consul` can help. In detail, I'll not repeat it here, please refer to my previous article!

Now that we understand how the naming system works, let's examine how to do inter-service communication based on that. 

### Communication Perspective

Inter-service communication is at the heart of all distributed systems. For nondistributed platforms, communication between processes can be done by `sharing memories`. But the communication between processes on different machines has always been based on the low-level message passing as offered by the underlying `network`. There are two widely used models for communications in distributed systems: `Remote Procedure Call(RPC)` and `Message-Oriented Middleware(MOM)`. 

- `Remote Procedure Call(RPC)`: is ideal for `client-server` applications and aims at hiding message-passing details from the client. The `HTTP` protocol is a typical `client-server` model, so `HTTP` requests can be thought of as a type of `RPC`.  In the `client-server` model, the client and server `must be online at the same time` to `exchange` information, so it's often called `synchronous communication`.  

- `Message-Oriented Middleware(MOM)`: is suitable for distributed applications, where the communication does not follow the strict pattern of a client-server model. In this model, there are three parties involved: the `message producer`, the `message consumer`, and the `message broker`. In this case, when the `message producer` publishes the message, the `message consumer` doesn't have to be online, the `message broker` will store the message until the consumer becomes available to receive it, so this model is also called `asynchronous communication`.

The `Service Fabric` supports `RPC` out of the box. In our `HealthMetrics` demo app, you can find the `RPC` calls and HTTP requests easily. For example, when the `BandActor` and the `DoctorActor` are created, the `RPC` call is used in the [`BandCreationService`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetrics.BandCreationService/Service.cs#L86) as follows: 

```csharp
private async Task CreateBandActorTask(BandActorGenerator bag, CancellationToken cancellationToken)
{
    // omit some code for simplicity
    ActorId bandActorId;
    ActorId doctorActorId;
    bandActorId = new ActorId(Guid.NewGuid()); 
    doctorActorId = new ActorId(bandActorInfo.DoctorId);
    IDoctorActor docActor = ActorProxy.Create<IDoctorActor>(doctorActorId, this.DoctorServiceUri);
    await docActor.NewAsync(doctorName, randomCountyRecord);
    IBandActor bandActor = ActorProxy.Create<IBandActor>(bandActorId, this.ActorServiceUri);
    await bandActor.NewAsync(bandActorInfo);
    // omit some code for simplicity
}
```

You can see the `ActorProxy` instance is created as the `RPC` client, and the `NewAsync` method of `BandActor` and `DoctorActor` service is called. For example, the [`NewAsync`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetrics.BandActor/BandActor.cs#L78) method of `BandActor` goes like this: 

```csharp
public async Task NewAsync(BandInfo info)
{
    await this.StateManager.SetStateAsync<CountyRecord>("CountyInfo", info.CountyInfo);
    await this.StateManager.SetStateAsync<Guid>("DoctorId", info.DoctorId);
    await this.StateManager.SetStateAsync<HealthIndex>("HealthIndex", info.HealthIndex);
    await this.StateManager.SetStateAsync<string>("PatientName", info.PersonName);
    await this.StateManager.SetStateAsync<List<HeartRateRecord>>("HeartRateRecords", new List<HeartRateRecord>()); // initially the heart rate records are empty list
    await this.RegisterReminders();

    ActorEventSource.Current.ActorMessage(this, "Band created. ID: {0}, Name: {1}, Doctor ID: {2}", this.Id, info.PersonName, info.DoctorId);
}
```

You can ignore the detailed content of this method for now, I will explain in a later section.

And when the `DoctorActor` service reports its status, it calls the `CountyService` endpoint with HTTP requests: 

```csharp 
public async Task SendHealthReportToCountyAsync()
{
    // omit some code
    await servicePartitionClient.InvokeWithRetryAsync(
        client =>
        {
            Uri serviceAddress = new Uri(
                client.BaseAddress,
                string.Format(
                    "county/health/{0}/{1}",
                    partitionKey.Value.ToString(),
                    id));

            HttpWebRequest request = WebRequest.CreateHttp(serviceAddress);
            request.Method = "POST";
            request.ContentType = "application/json";
            request.KeepAlive = false;
            request.Timeout = (int) client.OperationTimeout.TotalMilliseconds;
            request.ReadWriteTimeout = (int) client.ReadWriteTimeout.TotalMilliseconds;

            using (Stream requestStream = request.GetRequestStream())
            {
                using (BufferedStream buffer = new BufferedStream(requestStream))
                {
                    using (StreamWriter writer = new StreamWriter(buffer))
                    {
                        JsonSerializer serializer = new JsonSerializer();
                        serializer.Serialize(writer, payload);
                        buffer.Flush();
                    }

                    using (HttpWebResponse response = (HttpWebResponse) request.GetResponse())
                    {
                        ActorEventSource.Current.Message("Doctor Sent Data to County: {0}", serviceAddress);
                        return Task.FromResult(true);
                    }
                }
            }
        }
    );
    // omit some code
}
```

In the demo app, no `asynchronous communication` is used. But you can easily integrate the `message broker` middleware with the `Service Fabric`. This kind of microservice design is called `Event-Driven Architecture`. In the future, I will write another article about it, please keep watching my blog!

### Scalability Perspective

`Scalability` has become one of the most important design goals for developers of distributed systems. A system can be scalable, meaning that we can easily add more users and resources to the system without any noticeable loss of performance. In most cases, scalability problems in distributed systems appear as performance problems caused by the limited capacity of servers and networks. 

Simply speaking, there are two types of scaling techniques: `scaling up` and `scaling out`: 

- scaling up: is the process by which a machine is equipped with more and often more powerful resources(e.g., by increasing memory, upgrading CPUs, or replacing network modules) so that it can better accommodate performance-demanding applications. However, there are limits to how much you can scale up a single machine, and at some point, it may become more cost-effective to scale out. 
- scaling out: is all about extending a networked computer system with more computers and subsequently distributing workloads across the extended set of computers. There are basically only three techniques we can apply: `asynchronous communication`, `replication` and `Partitioning`. 

We already examined asynchronous communication in the last section. Next, let's take a deep look at the other two: 

- `Replication`:  is a technique, which `replicates` more components or resources, etc., across a distributed system. `Replication` not only increases `availability`; but also helps to balance the load between components, leading to `better performance`. In Service Fabric, no matter whether it is a stateless or stateful service, you can replicate the service across multiple nodes. As the workload of your application increases, Service Fabric will automatically distribute the load. But `replication` can only boost performance for `read` requests (which don't change data); if you need to optimize the performance for `write` requests (which change data), you need `Partitioning`. 

- `Partitioning`:  is an important scaling technique, which involves taking a component or other resource, splitting it into `smaller parts`, and subsequently spreading those parts across the system. Each partition only contains a subset of the entire dataset. This can help reduce the amount of data that needs to be processed and accessed by each partition,  which can lead to faster processing times and improved performance. In addition to reducing the size of the data set, partitioning can also improve concurrency and reduce contention.

In Service Fabric, each `partition` consists of a `replica set` with a single `primary` replica and multiple active `secondary` replicas. Service Fabric makes sure to distribute replicas of partitions across nodes so that secondary replicas of a partition do not end up on the same node as the primary replica, which can increase the `availability`. 

<img src="/images/partition-replication.png" title="partition and replicas" width="600px" height="400px"> 

The difference between a `primary` replica and a `secondary` replica is that the `primary` replica can handle both `read and write` requests, while the `secondary` replica can only handle `read` requests. Moreover, by default, the read requests are only handled by the `primary` replica, if you want to balance the read requests among all the `secondary` replicas, you need to set [`ListenOnSecondary`](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-lifecycle) at the application code level. You can set the number of replicas with [`MinReplicaSetSize`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetricsApplication/Scripts/HealthMetricsDeployment.ps1#L155) and [`TargetReplicaSetSize`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetricsApplication/Scripts/HealthMetricsDeployment.ps1#L158)

And when the client needs to call the service with multiple partitions, the client needs to generate a `ServicePartitionKey`; and send requests to the service with this partition key. For example, the `DoctorActor` sends the health report to the county service with multiple partitions as follows: 

```csharp
public async Task SendHealthReportToCountyAsync() {
    // omit some code 
    ServicePartitionKey partitionKey = new ServicePartitionKey(countyRecord.CountyId);
    ServicePartitionClient<HttpCommunicationClient> servicePartitionClient =
    new ServicePartitionClient<HttpCommunicationClient>(
        this.clientFactory,
        this.countyServiceInstanceUri,
        partitionKey);
    await servicePartitionClient.InvokeWithRetryAsync();
    // omit some code
}
```

You can see the partition key is generated based on the county id field. Service Fabric provides several different [partition schemas](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-concepts-partitioning). In this case, `ranged partitioning` is used, where the partition key is an `integer`. 

The county service specifies an integer range by a [`Low Key`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetricsApplication/Scripts/HealthMetricsDeployment.ps1#L34) ( set to 0) and [`High Key`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetricsApplication/Scripts/HealthMetricsDeployment.ps1#L35) (set to 57000). It also defines the number of partitions as [`PartitionCount`](https://github.com/baoqger/service-fabric-dotnet-data-aggregation/blob/master/HealthMetricsApplication/Scripts/HealthMetricsDeployment.ps1#L61) (set to 3). All the integer keys are evenly distributed among the partitions. So the partitions of the county service go as follows: 

<img src="/images/partition-key.png" title="partition key" width="400px" height="300px"> 

As I mentioned above, the county id is unique, we could then generate a hash code based on the id field, then modulus the key range, to finally get the partition key. Service Fabric runtime will direct the requests to the target node based on that partition key. 

### Summary

In this post, we quickly examined some interesting topics about Service Fabric. I have to admit that Service Fabric is a big project, what I examined here only covers a small portion of the entire system. Feel free to explore more!