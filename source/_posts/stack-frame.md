---
title: Understand stack memory management
date: 2020-08-19 04:54:00
tags:
---

### Table of Contents 
First I have to admit that memory management is a big( and important) topic in operating system. This post can't cover everything related to it. 

In this post, I want to share something interesting I learned about stack memory management. Especially understand how the stack frame of function call works, which's the most important part of stack memory. I will explain the mechanism in detail with examples and diagrams.

Briefly speaking, the contents of this post is:

- Memory layout of a process
- Stack memory contents
- CPU register related to stack memory management
- Stack frame of function call and return mechanism

To understand the concepts and machanisms deeply, a few assembly code will be shown in this post, you don't have to know assembly lauguage as an expert for reading this post, I will add comments for these assembly codes to make explain what's their functionalities.


### Stack Memory Management Basic

#### Memory Layout of Process

Before we talk about Stack memory management, it's necessary to understand `memory layout` of a process.

When you create a program, it is just some Bytes of data which is stored in the disk. When a program executes then it starts loading into memory and become a live entity. In a more specify way, some memory (virtual memory) is allocated to each process in execution for its usage by operating system. The memory assigned to a process is called process's `virtual address space`( short for VAS). And memory layout use diagrams to help you understand the concept of process virtual address space easily.

There are some awesome posts on the internet about [memory layout](https://www.includehelp.com/operating-systems/memory-layout-of-a-process.aspx). And the most two critical sections are: Stack and Heap. Simply speaking, `Stack` is the memory area which is used by each process to store the local variables, passed arguments and other information when a function is called. This is the topic of this post, you'll know more detailed in the following sections. While `Heap` segment is the one which is used for dynamic memory allocation, this is another more complex topic out of this post's scope. 

#### Stack Memory Basics
Stack Memory is just memory region in each process's virtual address space where `stack` data structure (Last in, first out) is used to store the data. As we mentioned above, when a new function call is invoked, then a frame of data is pushed to stack memory and when the function call returns then the frame is removed from the stack memory. This is **Stack Frame** which represent a function call and its argument data. And every function has its own stack frame.

Let's say in your program there are multiple functions call each other in order. For example, `main() -> f1() -> f2() -> f3()`, `main` function calls function one `f1()`, then function one calls function two `f2()`, finally function two calls function three `f3()`. So based on the Last in first out rule, the stack memory will be like as below: 

![stack-frame](/images/stack_frames.png)





### Terminology

### Procedure call

### Procedure return
