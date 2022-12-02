---
title: "External sorting: part one"
date: 2022-12-02 16:44:29
tags: algorithm, external, disk
---

### Introduction

This article will examine one interesting question which I came across when reading the book ***Programming Pearls***. This question simply goes like this: `How to sort a large disk file? The disk file has so much data that it cannot all fit into the main memory.` I considered this `algorithm` question for a while; but noticed that all the classic sorting algorithms, like [`Quick sort`](https://en.wikipedia.org/wiki/Quicksort) and [`merge sort`](https://en.wikipedia.org/wiki/Merge_sort), can't solve it easily. Then I did some research about it, and I will share what I learned in this article. I believe you can solve this question, after reading this article.

### Background of External Algorithm

Traditionally, computer scientists analyze the running time of an algorithm by counting the number of executed instructions, which is usually expressed as a function of the input size n, like the well-known [`Big O notation`](https://en.wikipedia.org/wiki/Big_O_notation). This kind of algorithm complexity analysis is based on the `Random access machine(RAM)` model, which defines the following assumptions:
- A machine with an unbounded amount of available memory;
- Any desired memory location can be accessed in unit time;
- Every instruction takes the same amount of time. 

outline

background of external algorithm

external sorting process 
    two ways
    multiple ways

Implementation two-way

implementation of multi-way

Performance evalution
