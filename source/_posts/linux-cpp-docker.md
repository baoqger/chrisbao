---
title: Use Docker container for local C++ development
date: 2020-07-28 20:42:21
tags: docker, c++, linux
---

### Why develop in Docker container?

`Docker` is the hottest technology to deploy and run the software. The key benefit of Docker is that it allows users to package an application with all of its dependencies into a standardized unit for software development. Compared with virtual machines, containers do not have high overhead and hence enable more efficient usage of the underlying system and resources. 

Besides deploying and running applications, Docker containers can also make your local development work easier, especially when you need to set up a specific environment with many dependencies. 

In my case, I have a project which is a C++ application running on the Linux platform. But on personal machines I'm running macOS and Windows systems, I didn't install Linux platform on my computer. Before I start working on this project, I need to fix the platform/environment issue. 

The first idea is, of course, using virtual-machines with VirtualBox and install the Linux system in it. This process will be time-consuming and tedious. So I decided to use Docker container to speed up the environment configuration step. 

I will share the experience step by step. The whole process is lightweight and quick, also can practice your Docker related skills.

### Create the Docker container

To build a Docker image, we need a `Dockerfile`, which is a text document(without a file extension) that contains the instructions to set up an environment for a Docker container. The Docker [official site](https://docs.docker.com/get-started/overview/) is the best place to understand those fundamental and important knowledge.

In my case the basic `Dockerfile` goes as following:

```docker
# Download base image ubuntu 20.04
FROM ubuntu:20.04

# Disable Prompt During Packages Installation
ARG DEBIAN_FRONTEND=noninteractive

# Update Ubuntu Software repository
RUN apt-get update && apt-get install -y \
    make \
    cmake \
    g++ \
    libncurses5-dev \
    libncursesw5-dev \
&& rm -rf /var/lib/apt/lists/*

CMD ["bash"]
```

**FROM** : the first part is the `FROM` command, which tells us what image to base this off of (as we know, Docker is following a multi-layer structure). In my case,  it's using the `Ubuntu:20.04` image, which again references a Dockerfile to automate the process. 

**ARG**: `ARG` instruction defines a variable that can be passed at build time to pass environment info. In this case, just to disable the console output during the following Linux package installation process. 

**RUN**: the next set of calls are the `RUN` commands. This `RUN` instruction allows you to install your application and packages for it. `RUN` executes commands in a new layer and creates a new layer by committing the results. Usually, you can find several `RUN` instructions in a Dockerfile. In this case, I want to install the C++ compiler and build tools (and some other specific dependency packages for development) which is not available in the Ubuntu base image.

**CMD**: the last instruction `CMD` allows you to set a default command, which will be executed when you run the container without specifying a command. If Docker container runs with a command, this default one will be ignored. 

With this `Dockerfile`, we can build the image with the next Docker command:

```docker
docker build -t linux-cpp .
```

This will build the desired Docker image tagged as `linux-cpp`. You can list(find) this new image in your system with command `docker images`: 

``` txt
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
linux-cpp           latest              5463808c4488        8 days ago          320MB
```

Now you can run the docker container with the newly build `linux-cpp` image:

```
docker run -it --name cpp-dev --rm linux-cpp
```

### 









