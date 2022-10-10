---
title: "Understand userland heap memory allocation: part three - free chunk"
date: 2022-10-10 16:27:46
tags: Linux, memory layout, heap, gdb
---

### Introduction

In the last [article](https://organicprogrammer.com/2022/09/08/userland-memory-allocation-two/),  we investigated how the allocated chunks are aligned and stored in the heap. This article continues to examine how to free a chunk of memory and how the freed chunks are stored in the heap. 

### Hands-on demo

简单点儿

1. 不是直接返回给os，那样太费性能了，而是用一个linked list来store
2. demo 下，free了之后，进入tcache bins，以及之后的reuse，demo的部分就可以了。
下面做一些讨论和研究，关于为什么用了这么多的bins
2. 稍微延申一点，为什么用linked list来做这个事情很合适 => 内存地址不连续
3. 引出各种bins，总结下各种bins存在价值 => linked list的index是它的不足点，所以需要不同的bin来提高性能
   tcache_bins其实是用的是hashtable的一个常见实现：seperate chain => 它的复杂是常数

   这块应该对数据结构和算法进行深化




bins => 忽略细节 linked list to store freed chunks for reusing
bins的内容 => linked list 支持不连续内存单元
展示top section的变化 ==> 从零开始
展示 bins的变化 ==> free and reuse