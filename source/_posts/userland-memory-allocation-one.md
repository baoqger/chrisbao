---
title: userland-memory-allocation-one
date: 2022-10-05 20:31:11
tags:
---

### Introduction

In my eyes, compared with developing applications with high-level programming languages, one of the biggest differences for system programming with low-level languages like C and C++, is you have to manage the memory by yourself. So you call APIs like `malloc`, and `free` to allocate the memory based on your need and release the memory when the resource is no longer needed. It is not only one of the most frequent causes of [bugs](https://developers.redhat.com/articles/2021/11/01/debug-memory-errors-valgrind-and-gdb) in system programming; but also can lead to many [security issues](https://en.wikipedia.org/wiki/Memory_safety). 

It's not difficult to understand the correct usage of APIs like `malloc`, and `free`. But have you ever wondered how they work, for example: 

- When you call `malloc`, does it trigger system calls and delegate the task to the kernel or there are some other mechanisms? 
- When you call `malloc(10)` and try to allocate 10 bytes of heap memory, how many bytes of memory do you get? 10 bytes or more?
- When the memory is allocated, where exactly the heap objects are located?
- When you call `free`, is the memory directly returned to the kernel? 

This article will try to answer these questions. 

Note that memory is a super complex topic, so I can't cover everything about it in one article (In fact, what is covered in this article is very limited). This article will focus on `userland memory(heap) allocation`. 

### Background

#### Process virtual memory

Every time we start a program, a memory area for that program is reserved, and that's `process virtual memory` as shown in the following image: 

<img src="/images/process-memory-address.png" title="process virtual memory" width="400px" height="300px">

You can note that each process has one **invisible** memory segment containing kernel codes and data structures. This invisible memory segment is important; since it's directly related to `virtual memory`, which is employed by the kernel for memory management. Before we dive into the other different segments, let's understand virtual memory first. 

#### Virtual memory technique

<img src="/images/virtual-memory-technique.png" title="virtual memory technique" width="600px" height="400px">

Why do we need virtual memory? Without virtual memory, applications would need to manage their physical memory space, coordinating with every other process running on the computer. Virtual memory leaves that management to the kernel by creating the maps that allow translation between virtual and physical memory. So virtual memory is a service provided by the kernel in the form of abstraction. We can also realize [process isolation](https://en.wikipedia.org/wiki/Process_isolation) based on virtual memory to enhance security. 

Virtual memory is out of this article's scope, if you're interested, please take a look at the core techniques: [paging](https://en.wikipedia.org/wiki/Memory_paging) and [swapping](https://linuxhint.com/linux-memory-management-swap-space/). 

Process Memory Layout.
Basic virtual memory introduction.
Detailed Memory layout => source of various sections <== memory allocation at compile time(static allocation of memory) and runtime time(dynamic allocation of memory). => ELF file => loader
*Consider multiple threads layout
Userland memory allocation ==> Key
    allocation memory in user space



analysis based on malloc
    various data structures: bins and chunks
    通过例子反推实现点
        alignment
        P bit
        big/little endian
        gdb command
        bins => 忽略细节 linked list to store freed chunks for reusing
        bins的内容 => linked list 支持不连续内存单元
        展示top section的变化 ==> 从零开始
        展示 bins的变化 ==> free and reuse
