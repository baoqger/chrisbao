---
title: "Understand userland heap memory allocation: part two - experiment"
date: 2022-09-08 15:43:57
tags: Linux, memory layout, heap, gdb
---

### Introduction

The previous [article](https://organicprogrammer.com/2022/08/05/userland-memory-allocation-one/) gave a general overview of memory management. The story goes on. In this section, let's break into the heap memory to see how it works basically. 

### Memory allocator

We need to first understand some terminology in the memory management field: 

- mutator: the program that modifies the objects in the heap, which is simply the user application. But I will use the term `mutator` in this article. 
- allocator: the `mutator` doesn't allocate memory by itself, it delegates this generic job to the `allocator`. At the code level, the `allocator` is generally implemented as a library. The detailed allocation behavior will be fully determined by the implementations, in this article I will focus on the memory allocator in the library of `glibc`.  

<img src="/images/memo-allocator.png" title="memory allocator" width="400px" height="300px">


Note based on malloc in glibc


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
