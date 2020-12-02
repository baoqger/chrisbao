---
title: Service registration and discovery in Golang Cloud-Native microservice with Consul and Docker
date: 2020-11-16 09:22:23
tags: microservice, service registration, service discovery, Consul, Golang, Cloud-Native, Docker
---

### Background

In this post, I will give a real demo application to show how to do service registration and discovery in `Cloud-Native` microservice architecture based on `Consul` and `Docker`. And the service is developed in `Golang` language.

It will cover the following technical points:
- Integrate Consul with Golang application for service registration
- Integrate Consul with Golang application for service discovery
- Configure and Run the microservices with Docker(docker-compose)

As you can see, this post will cover several critical concepts and interesting tools. I will a quick and brief introduction of them. 

- Cloud-Native: this is another buzzword in the software industry. One of the key attributes of Cloud-Native application is `containerized`. To be considered cloud native, an application must be `infrastructure agnostic` and use containers. Containers provide applications the ability to run as a stand-alone environment able to move in and out of the cloud and have no dependencies on any certain cloud provider. 

- Service Registration and Service Discovery: in the microservices application, each service needs to call other services. In order to make a request, your service needs to know the network address of a service instance. In a cloud-based microservices application, the network location is dynamically assigned. So your application needs a service discovery mechanism. On the other hand, the service registry acts as a database storing the available service instances.

- Consul: Consul is the tool we used in this demo application for service registry and discovery. Consul is a member in `CNCF(Cloud Native Computing Foundation)`. I will try to write a post to analyze its source code in the future.
  
- Docker-compose: is a tool to run multi-container applications on Docker. It allows different container can communicate with each other. In this post, I will show you how to use it as well. 



consul service registry

consul service discovery

consul service discovery multi instances cases: health check and load balance effect

consul CONSUL_HTTP_ADDR source code read

consul service discovery related package


