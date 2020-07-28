---
title: Use Docker container for local C++ development
date: 2020-07-28 20:42:21
tags: docker, c++, linux
---

### Why develop in Docker container?

`Docker` is the hottest technology to deploy and run the software. The key benefit of Docker is that it allows users to package an application with all of its dependencies into a standardized unit for software development. Compared with virtual machines, containers do not have high overhead and hence enable more efficient usage of the underlying system and resources. 

Besides deploying and running applications, Docker containers can also make your local development work easier, especially when you need to setup a specific environment with many dependencies. 

In my case, I have a project which is a C++ application running on Linux platform. But on personal machines I'm running MacOs and Windows systems, I didn't install Linux platform on my computer. Before I start working on this project, I need to fix the platform/environment issue. 

The first idea is, of course, using virtual-machines with VirtualBox and install Linux system in it. This process will be time-consuming and tedious. So I decided to use Docker container to speed up the environment configuration step. 

I will share the experience step by step. The whole process is lightweight and quick, also can practise your Docker related skills.

### Create the Docker container

