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
    void *c = malloc(1033);
    void *d = malloc(1033);
    free(c);
    getchar();
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

You can see the changes: the allocated chunk is marked as a `Free chunk (tcache)` and pointer `fd` is set. 

The `tcache` is one kind of `bins` provided by `glibc`. The gdb `pwndbg` plugin allows you to check the content of `bins` by running command `bins` as follows:

<img src="/images/heap-demo-bins.png" title="pwndbg 2" width="300px" height="200px">

Note that the freed chunk(at 0x5555555592a0) is inserted into `tcache bins` as the liked list header.  

Moreover, there 5 types of bins: `small bins`, `large bins`, `unsorted bins`, `fast bins` and `tcache bins`. I will examine them in the following section. 

According to the definition, after the second `malloc(100)` is called, the `allocator` should reuse the freed chunk in the `bins`. The following image can prove this:  

<img src="/images/heap-demo-reuse-100.png" title="pwndbg 3" width="300px" height="200px">

The freed chunk at 0x555555559290 is in use again and after the chunk is removed from the linked list all `bins` are empty. All right! 
### Recycling memory with bins

Next, I want to spend a little bit of time examining why we need `bins` and how `bins` optimize chunk allocation and free. 

<img src="/images/naive-free-list.png" title="pwndbg 4" width="400px" height="300px">

If the `allocator` keeps track of all the freed chunks in a linked list. It will take `O(N)` time for the allocator to find a freed chunk with fit size by traversing from the head to the tail. If the `allocator` wants to keep the chunks in order, then at least  `O(NlogN)` time is needed to sort the list by size. This slow allocation would have a bad impact on the overall performance of programs. Next, let's see how bins optimize this process. In summary, the optimization is done on the following three aspects:

- High-performance data structure
- fast-turnaround queue
- per-thread cache without lock contention

Among the 5 types of `bins`, `small bins` and `large bins` behave similarly. At the [code level](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1686), they are defined as follows:

```c
#define NBINS             128

typedef struct malloc_chunk* mchunkptr;

mchunkptr bins[NBINS * 2 - 2];
```

So they are an array of linked lists and each linked list(or bin) stores chunks that are all the same fixed size. The difference is the `small bin` tracks freed chunks less than 1024 bytes while the `large bin` is for bigger chunks.  `small bins` can be represented with the following image while `large bins` are quite similar: 

<img src="/images/small-bins-index.png" title="pwndbg 4" width="600px" height="400px">

The `glibc` provides a [function](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1686) to calculate the `index` of the corresponding small(or large) bin in the array based on the requested `size`. Since the `index` operation of the [array](https://en.wikipedia.org/wiki/Array_(data_structure)) is in `O(1)` time. Moreover, each bin contains chunks of the same size, so it can also take `O(1)` time to unlink the first chunk from the list. As a result, the entire allocation time is optimized to  `O(1)`. 

The other types of bins all use a similar data structure to speed up the process, I will not show all of them here, please explore by yourself! 



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