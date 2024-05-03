---
title: "Diagnose Memory Leaks in .NET Applications with WinDbg: A Hands-on Approach"
date: 2024-05-01 10:43:54
tags: dotnet, GC, memory leak, WinDbg
---

### Introduction

In this article, I will share what I learned about `.NET memory management` by troubleshooting a memory leak issue for a .NET service running in a real production environment. After reading this post, you can understand more about the following items:

- `.NET Memory Management` and `.NET Garbage Collection`
- Diagnose memory usage with `WinDbg`

Previously I once published [articles](https://organicprogrammer.com/2022/08/05/userland-memory-allocation-one/) about memory allocation of native applications written in C on Linux systems. I highly recommend you read both articles and compare what's the difference, this can deepen your understanding of memory-related techniques.  

### Background

Firstly, let me explain the background: the problematic .NET service is running on the [`Azure Service Fabric`](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-azure-clusters-overview), which consists of a cluster of [Windows Server](https://en.wikipedia.org/wiki/Windows_Server). If you are not familiar with Service Fabric, please refer to my [previous article](https://organicprogrammer.com/2023/03/20/guide-to-service-fabric-architecture/), which explains how to develop microservices based on it. The problem is that the memory usage of this .NET service reaches roughly `1.5GB`, which is 4 times higher than the average usage. Generally speaking, on the cloud more resource usage means higher cost. Next, let's diagnose what is happening here. But before we roll up our sleeves to debug this issue, let's first review how .NET memory works at a theoretical level.

### .NET Memory Management

##### .NET Runtime

Unlike native software written directly to the operating system's APIs, programs written with `.NET` are called `managed` applications because they depend on a runtime and framework that manages many of their vital tasks and ensures a basic safe operating environment. In `.NET`, runtime refers to the `Common Language Runtime(CLR)` and the framework is `.NET Framework` or `.NET Core`. And the .NET application is built on top of them. 

<img src="/images/CLR.png" title=".NET and CLR" width="600px" height="400px">

`CLR` works as a virtual execution engine for .NET applications, and it is a very complex topic that is out of the scope of this article, so we can't dive into the details. But one of the crucial functionalities it provides is just: `memory management`. 

##### .NET GC

When you're working on system programming with C or C++, you need to manage the memory by calling standard library's APIs like `malloc` and `free` to allocate and release the memory. But on the .NET framework, your life will be easier since the .NET memory manager does this task for you. When the program creates a new object, the memory manager will allocate the memory for you. Easy, right? But that's only a trivial part of the memory manager, the complicated task is how and when the memory manager can decide to collect the memory. Because of this, the .NET memory manager has another name: `.NET Garbage Collector`. 

The .NET GC uses two Win32 functions `VirtualAlloc` and `VirtualFree` for allocating a segment of memory and releasing segments back to the operating system respectively. The entire process can be divided into the following three phases: 

<img src="/images/gcphases.png" title=".NET and CLR" width="600px" height="400px">


- **Marking Phase**: A list of `live` objects is created during the marking phase. This is done by starting from a set of references called the `GC roots`. GC marks these root objects as `live` and then looks at any object that they reference and marks these as being `live` too. And the GC continues the search iteratively. This process is also called `reachability analysis`. All of the objects that are not on the reachable list are potentially released from the memory. 

<img src="/images/reachability.png" title="GC roots" width="600px" height="400px">

- **Relocating Phase**: The references of all the reachable objects are updated so that they point to the new location. The purpose of the relocating phase is to optimize memory usage by compacting the live objects closer together. This helps reduce `memory fragmentation` and improves memory allocation efficiency. 

- **Compacting Phase**: The memory space occupied by the dead objects is released and the live objects are moved to the new location. 

GC algorithm is a complex topic, the above description only touches the surface of it. You can explore [more](https://www.red-gate.com/simple-talk/development/dotnet-development/understanding-garbage-collection-in-net/) by yourself. In the following sections, I will add extra information when necessary. 

### WinDbg 

sos extension

### Diagnose the memory leak

!address -summary different segments explanation, committed memory
!eeheap -stat
!finalizequeue


.net GC, memory leak, GC algorithm, procdump for memory dump
WinDbg command => address, unknown, commit, finalizequeue, managed vs unmanaged, CLR, virtual memory

