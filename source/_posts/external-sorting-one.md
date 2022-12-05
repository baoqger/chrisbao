---
title: "External sorting: part one"
date: 2022-12-02 16:44:29
tags: algorithm, external, disk
---

### Introduction

This article will examine one interesting question which I came across when reading the book ***Programming Pearls***. This question simply goes like this: `How to sort a large disk file? The disk file has so much data that it cannot all fit into the main memory.` I considered this `algorithm` question for a while; but noticed that all the classic sorting algorithms, like [`Quick sort`](https://en.wikipedia.org/wiki/Quicksort) and [`merge sort`](https://en.wikipedia.org/wiki/Merge_sort), can't solve it easily. Then I did some research about it, and I will share what I learned in this article. I believe you can solve this problem as well, after reading this article.

### Background of External Algorithm

Traditionally, computer scientists analyze the running time of an algorithm by counting the number of executed instructions, which is usually expressed as a function of the input size n, like the well-known [`Big O notation`](https://en.wikipedia.org/wiki/Big_O_notation). This kind of algorithm complexity analysis is based on the `Random access machine(RAM)` model, which defines the following assumptions:
- A machine with an unbounded amount of available memory;
- Any desired memory location can be accessed in unit time;
- Every instruction takes the same amount of time. 

<img src="/images/RAM-Model.png" title="Random access machine model" width="300px" height="200px">

This model works well when the amount of memory that the algorithm needs to use is smaller than the amount of memory in the computer running the code. 

But in the real world, some applications need to process data that are too large to fit into a computer's main memory at once. And the algorithms used to solve such problems are called `external memory algorithms` or `disk-based algorithms`; since the input data are stored on an [`external memory` storage device](https://en.wikipedia.org/wiki/External_storage).

Instead of the `RAM` model, the `external memory algorithm` is analyzed based on the `external memory model`. The model consists of a CPU processor with an internal memory of bounded size, connected to an unbounded external memory. One I/O operation consists of moving a block of contiguous elements from external to internal memory. 

<img src="/images/external-model.png" title="external memory model" width="400px" height="300px">

Compared with the main memory, the access speed of external memory is slower by [several orders of magnitude](https://en.wikipedia.org/wiki/Memory_hierarchy), even though modern storage techniques such as [`SSD`](https://en.wikipedia.org/wiki/Solid-state_drive) are already adopted. Thus, for external algorithms, the bottleneck of the performance is `disk IO` speed instead of the CPU cycles. 




outline

background of external algorithm

external sorting process 
    two ways
    multiple ways

Implementation two-way

implementation of multi-way

Performance evalution
