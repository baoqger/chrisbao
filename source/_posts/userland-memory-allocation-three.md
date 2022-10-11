---
title: "Understand userland heap memory allocation: part three - free chunk"
date: 2022-10-10 16:27:46
tags: Linux, memory layout, heap, gdb
---

### Introduction

In the last [article](https://organicprogrammer.com/2022/09/08/userland-memory-allocation-two/),  we investigated how the allocated chunks are aligned and stored in the heap. This article continues to examine how to free a chunk of memory and how the freed chunks are stored in the heap. 

### Hands-on demo

Let's continue debugging the demo code shown in the last article: 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <malloc.h>

int main(int argc, char *argv[]) {
    char *a = (char*)malloc(100);
    strcpy(a, "AAAABBBBCCCCDDDD");
    free(a);
    char *b = (char*)malloc(100);
    free(b);
    return 0;
}
```
Previously, we allocated a chunk of memory and put data in it. The next line will free this chunk. Before we run the instruction and show the demo result, let's discuss the theory first. 

The freed chunk will not be returned to the kernel immediately after the `free` is called. Instead, the heap `allocator` keeps track of the freed chunks in a [`linked list`](https://en.wikipedia.org/wiki/Linked_list) data structure. So the freed chunks in the linked list can be reused when the application requests new allocations again. This can decrease the performance overhead by avoiding too many system calls. 

If the `allocator` stores all the freed chunks together in a long linked list, it would work but the performance would be slow. Instead, the `glibc` maintains a series of freed linked lists called `bins`, which can speed up the allocations and frees. We will examine how `bins` work later.  

It is worth noting that each free chunk needs to store `pointers` to other chunks to form the linked list. That's what we discussed in the last section, there're two points in the `malloc_chunk` structure: `fd` and `bk`, right? Since the `user data` region of the freed chunk is free for use by the `allocator`, so it repurposes the `user data` region as the place to store the pointer. 

Based on the above description, the following picture illustrates the exact structure of a freed chunk:

```
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of previous chunk, if freed                  | 
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of chunk, in bytes                     |A|M|P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             pointer to the next freed chunk                   |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             pointer to the previous freed chunk               |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            .                                                               .
            .                           ......                              .
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of chunk                                     |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```

Now step over one line in `gdb` and check chunks in the heap as follows: 

<img src="/images/heap-demo-heap-free.png" title="pwndbg 1" width="300px" height="200px">

You can see the changes: the allocated chunk is marked as `Free chunk (tcache)`

简单点儿

1. 不是直接返回给os，那样太费性能了，而是用一个linked list来store
2. demo 下，free了之后，进入tcache bins，以及之后的reuse，demo的部分就可以了。
下面做一些讨论和研究，关于为什么用了这么多的bins
3. 稍微延申一点，为什么用linked list来做这个事情很合适 => 内存地址不连续
malloc_state 数据结构的repurpose

4. 引出各种bins，总结下各种bins存在价值 => linked list的index是它的不足点，所以需要不同的bin来提高性能
   tcache_bins其实是用的是hashtable的一个常见实现：seperate chain => 它的复杂是常数

   这块应该对数据结构和算法进行深化




bins => 忽略细节 linked list to store freed chunks for reusing
bins的内容 => linked list 支持不连续内存单元
展示top section的变化 ==> 从零开始
展示 bins的变化 ==> free and reuse