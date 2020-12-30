---
title: Load balancing in Golang Cloud-Native microservice with Consul and Fabio
date: 2020-12-30 09:56:57
tags: microservice, Load balancing, Consul, Fabio Golang, Cloud-Native, Docker
---

### Background

In the [last post](https://baoqger.github.io/2020/11/16/golang-service-discovery-consul/), I show you how to do service discovery in a Golang Cloud-Native microservice application based on Consul and Docker with a real demo. In that demo, the simple `helloworld-server` service registers itself in Consul and the `helloworld-client` can discover the dynamic address of the service via Consul. But the previous demo has some limitations, as I mentioned in the last post, in the real world microservice application, each service may have multiple instances to handle the network requests. 

In this post, I will expand the demo to show you how to do `load balancing` when multiple instances of one service are registered in Consul. 

Continue with the last post, the new demo will keep using Cloud-Native way with `Docker` and `Docker-compose`.

### Introduction of `Fabio` for load balancing 

To do load balancing for Consul, there are several strategies are recommended from the Consul official document. In this post I choose to use [Fabio](https://github.com/fabiolb/fabio). 

> Fabio is an open source tool that provides fast, modern, zero-conf load balancing HTTP(S) and TCP router for services managed by Consul. Users register services in Consul with a health check and fabio will automatically route traffic to them. No additional configuration required.

Fabio is an interesting project, it realizes loading balancing based on the `tag` information of service registration in Consul. 

Users register a service with a tag beginning with `urlprefix-`, like:

```
urlprefix-/my-service
```

Then when a request is made to fabio at `/my-service`, fabio will automatically route traffic to a healthy service in the cluster. I will show you how to do it in the following demo and also have simple research on how Fabio realize this load balancing strategy by reviewing the source code. 

### Fabio Load balancing demo

Firstly, all the code and config file shown in this post can be found in this [github repo](https://github.com/baoqger/service-discovery-demo), please `git checkout` the `load-balancing` branch for this post's demo. 

#### Server side
For the `helloworld-server`, there are two changes:
  - First, each service instance should have an unique `ID`;
  - Second, add `Tags` for service registration and the tag should follow the rule of `Fabio`. 

Ok, let's check the new version code. 

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"strconv"

	consulapi "github.com/hashicorp/consul/api"
)

func main() {
	serviceRegistryWithConsul()
	log.Println("Starting Hello World Server...")
	http.HandleFunc("/helloworld", helloworld)
	http.HandleFunc("/check", check)
	http.ListenAndServe(getPort(), nil)
}

func serviceRegistryWithConsul() {
	config := consulapi.DefaultConfig()
	consul, err := consulapi.NewClient(config)
	if err != nil {
		log.Println(err)
	}

	port, _ := strconv.Atoi(getPort()[1:len(getPort())])
	address := getHostname()
	/* Each service instance should have an unique serviceID */
	serviceID := fmt.Sprintf("helloworld-server-%s:%v", address, port)
	/* Tag should follow the rule of Fabio: urlprefix- */
	tags := []string{"urlprefix-/helloworld"}

	registration := &consulapi.AgentServiceRegistration{
		ID:      serviceID,
		Name:    "helloworld-server",
		Port:    port,
		Address: address,
		Tags:    tags, /* Add Tags for registration */
		Check: &consulapi.AgentServiceCheck{
			HTTP:     fmt.Sprintf("http://%s:%v/check", address, port),
			Interval: "10s",
			Timeout:  "30s",
		},
	}

	regiErr := consul.Agent().ServiceRegister(registration)

	if regiErr != nil {
		log.Printf("Failed to register service: %s:%v ", address, port)
	} else {
		log.Printf("successfully register service: %s:%v", address, port)
	}
}

func helloworld(w http.ResponseWriter, r *http.Request) {
	log.Println("helloworld service is called.")
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Hello world.")
}

func check(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Consul check")
}

func getPort() (port string) {
	port = os.Getenv("PORT")
	if len(port) == 0 {
		port = "8080"
	}
	port = ":" + port
	return
}

func getHostname() (hostname string) {
	hostname, _ = os.Hostname()
	return
}

```
##### **`server.go`**

The changes are at Line 30, 32 and 40 and comments are added there to explain the purpose of the change. Simply speaking, now each service instance registers itself with a unique ID, which is consisted of the basic service name (helloworld-server in this case) and the dynamic address. Also, we add `urlprefix-/helloworld` **Tags** for each registration. `urlprefix-` is the default config of Fabio, you can set customized prefix if needed. Based on this **Tags**, Fabio can do automatic load balancing for the `/helloworld` endpoint. 

That's all for the code change for server side. Let's review the changes for the client. 

#### Client side

```golang
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"time"

	consulapi "github.com/hashicorp/consul/api"
)

var url string

/*
For load balancing, run fabioLoadBalancing();
For simple service discovery, run serviceDiscoveryWithConsul();
*/
func main() {
	fabioLoadBalancing()
	fmt.Println("Starting Client.")
	var client = &http.Client{
		Timeout: time.Second * 30,
	}
	callServerEvery(10*time.Second, client)
}

/* Load balancing with Fabio */
func fabioLoadBalancing() {
	address := os.Getenv("FABIO_HTTP_ADDR")
	url = fmt.Sprintf("http://%s/helloworld", address)
}

/* Service Discovery with Consul */
func serviceDiscoveryWithConsul() {
	config := consulapi.DefaultConfig()
	consul, error := consulapi.NewClient(config)
	if error != nil {
		fmt.Println(error)
	}
	services, error := consul.Agent().Services()
	if error != nil {
		fmt.Println(error)
	}

	service := services["helloworld-server"]
	address := service.Address
	port := service.Port
	url = fmt.Sprintf("http://%s:%v/helloworld", address, port)
}

func hello(t time.Time, client *http.Client) {
	response, err := client.Get(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	body, _ := ioutil.ReadAll(response.Body)
	fmt.Printf("%s. Time is %v\n", body, t)
}

func callServerEvery(d time.Duration, client *http.Client) {
	for x := range time.Tick(d) {
		hello(x, client)
	}
}

```
##### **`client.go`**

Previously, we need to run `serviceDiscoveryWithConsul` to discover the service address to call. Now since we have `Fabio` working as the load balancer, so we send the request to `Fabio` and our request will be distributed to the service instance by `Fabio`.

This part of logic is implemented inside the following method: 

```golang 
/* Load balancing with Fabio */
func fabioLoadBalancing() {
	address := os.Getenv("FABIO_HTTP_ADDR")
	url = fmt.Sprintf("http://%s/helloworld", address)
}
```
To get the address of the Fabio service, we need to config it as an environment variable, which will be set in the `yml` file of Docker-compose. Let's review the new `yml` file now. 

#### Docker-compose config

```yml
version: '2'

services: 
  consul:
    image: consul:0.8.3
    ports:
      - "8500:8500"
    networks:
      - my-net
      
  helloworld-server:
    build:
      context: .
      dockerfile: server/Dockerfile
    image: helloworld-server:1.0.2 # upgrade to v1.0.2
    environment: 
      - CONSUL_HTTP_ADDR=consul:8500
    depends_on:
      - consul
    networks:
      - my-net

  helloworld-client:
    build:
      context: .
      dockerfile: client/Dockerfile
    image: helloworld-client:1.0.2 # upgrade to v1.0.2
    environment: 
      - CONSUL_HTTP_ADDR=consul:8500
      - FABIO_HTTP_ADDR=fabio:9999 # environment variable for fabio service address
    depends_on:
      - consul
      - helloworld-server
    networks:
      - my-net

  fabio: # add a new service: Fabio as load balancer
    image: fabiolb/fabio:latest
    environment: 
      - registry_consul_addr=consul:8500 # environment variable for consul service address
      - proxy_strategy=rr # environment variable for load balancing strategy. rr is round-robin      
    ports:
      - "9998:9998"
      - "9999:9999"
    depends_on:
      - consul  
    networks:
      - my-net 

networks:
  my-net:
    driver: bridge
```
##### **`docker-compose.yml`**

There several changes in this `yml` config file:
- Add a new service `Fabio`. As mentioned above Fabio is a zero-conf load balancing, which can simply run as a docker container. This is so convenient and totally matches Cloud-Native style. The two environment variables: `registry_consul_addr` and `proxy_strategy`, are set to define the Consul's address and the round-robin strategy. 
- Set the `FABIO_HTTP_ADDR` environment variable for the client. This is what we mentioned in the last section, which allows `client.go` to get Fabio service address and send requests. 
- Upgrade two docker images to **v1.0.2**.  

#### Demo

It's time to run the demo! Suppose you have all the docker images build in your local machine, then run the following command: 

```
docker-compose up --scale helloworld-server=3
```

This command has an important tip about Docker-compose: how to run multiple instances of certain service. In our case, we need multiple instances of `helloworld-server` for load balancing. Docker-compose supports this functionality with `--scale` option. For the above command, 3 instances of helloworld-server will be launched.

You can see the demo's result in the following image: 

![load-balancing](/images/load-balancing.png)

The client repeatedly and periodically sends the request and each request is distributed by Fabio to one of the three instances in round-robin style. Just what we expect! 
