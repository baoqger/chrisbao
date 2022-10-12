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

The `allocator` could store all the freed chunks together in a long linked list, this would work but the performance would be slow. Instead, the `glibc` maintains a series of freed linked lists called `bins`, which can speed up the allocations and frees. We will examine how `bins` work later.  

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

You can see the changes: the allocated chunk is marked as a `Free chunk (tcache)` and pointer `fd` is set(which indicates this freed chunk is inserted into a linked list). 

The `tcache` is one kind of `bins` provided by `glibc`. The gdb `pwndbg` plugin allows you to check the content of `bins` by running command `bins` as follows:

<img src="/images/heap-demo-bins.png" title="pwndbg 2" width="300px" height="200px">

Note that the freed chunk(at 0x5555555592a0) is inserted into `tcache bins` as the liked list header.  

Note that there 5 types of bins: `small bins`, `large bins`, `unsorted bins`, `fast bins` and `tcache bins`. If you don't know, don't worry I will examine them in the following section. 

According to the definition, after the second `malloc(100)` is called, the `allocator` should reuse the freed chunk in the `bins`. The following image can prove this:  

<img src="/images/heap-demo-reuse-100.png" title="pwndbg 3" width="300px" height="200px">

The freed chunk at 0x555555559290 is in use again and all `bins` are empty after the chunk is removed from the linked list. All right! 
### Recycling memory with bins

Next, I want to spend a little bit of time examining why we need `bins` and how `bins` optimize chunk allocation and free. 

If the `allocator` keeps track of all the freed chunks in a long linked list. The time complexity is `O(N)` for the allocator to find a freed chunk with fit size by traversing from the head to the tail. If the `allocator` wants to keep the chunks in order, then at least  `O(NlogN)` time is needed to sort the list by size. This slow process would have a bad impact on the overall performance of programs. That's the reason why we need bins to optimize this process. In summary, the optimization is done on the following two aspects:

- High-performance data structure
- Per-thread cache without lock contention

#### High-performance data structure

Take the `small bins` and [`large bins`](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1686) as a reference, they are defined as follows:

```c
#define NBINS             128

typedef struct malloc_chunk* mchunkptr;

mchunkptr bins[NBINS * 2 - 2];
```

They are defined together in an array of linked lists and each linked list(or bin) stores chunks that are all `the same fixed size`. From `bins[2] to bins[63]` are the `small bins`, which track freed chunks less than 1024 bytes while the `large bins` are for bigger chunks.  `small bins` and `large bins` can be represented as a [`double-linked list`](https://en.wikipedia.org/wiki/Doubly_linked_list) shown below: 

<img src="/images/small-bins-index.png" title="pwndbg 4" width="600px" height="400px">

The `glibc` provides a [function](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1686) to calculate the `index` of the corresponding small(or large) bin in the array based on the requested `size`. Since the `index` operation of the [array](https://en.wikipedia.org/wiki/Array_(data_structure)) is in `O(1)` time. Moreover, each bin contains chunks of the same size, so it can also take `O(1)` time to insert or remove one chunk into or from the list. As a result, the entire allocation time is optimized to  `O(1)`. 

`bins` are `LIFO(Last In First Out)` data structure. The insert and remove operations can be illustrated as follows: 
 
<img src="/images/LIFO-linked-list.png" title="pwndbg 4" width="600px" height="400px">

Moreover, for `small bins` and `large bins`, if the neighbors of the current chunk are free, they are `merged` into a larger one. That's the reason we need a `double-linked list` to allow running fast traverse both forward and backward. 

Unlike `small bins` and `large bins`, `fast bins` and `tcache bins` chunks are `never merged` with their neighbors. In practice, the glibc `allocator` doesn't set the `P` special flag at the start of the next chunk. This can avoid the overhead of merging chunks so that the freed chunk can be immediately reused if the same size chunk is requested. Moreover, since `fast bins` and `tcache bins` are never merged, they are implemented as a [`single-linked list`](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1678).

This can be proved by running the second `free` method in the demo code and checking the chunks in the heap as follows: 

<img src="/images/heap-demo-heap-free.png" title="pwndbg 1" width="300px" height="200px">

First, the `top` chunk's size is still `0x20d01` rather than `0x20d00`, which indicates the `P` bit is equal to 1. Second, the `Free chunk` only has one pointer: `fd`. If it's in a double-linked list, both `fd` and `bk` should point to a valid address. 

#### Per-thread cache without lock contention

The letter `t` in `tcache bins` represents the `thread`, which is used to optimize the performance of multi-thread programs. In multi-thread programming, the most common way solution to prevent the [`race condition`](https://en.wikipedia.org/wiki/Race_condition) issue is using the [`lock or mutex`](https://en.wikipedia.org/wiki/Lock_(computer_science)). Similarly, The `glibc` maintains a `lock` in the data structure for each heap. But this design comes with a performance cost: [`lock contention`](https://en.wikipedia.org/wiki/Lock_(computer_science)#:~:text=lock%20contention%3A%20this%20occurs%20whenever,lock%20held%20by%20the%20other.), which happens when one thread attempts to acquire a lock held by another thread. This means the thread can't do any tasks. 

`tcache bins` are per-thread bins. This means if the thread has a chunk on its `tcache bins`, it can serve the allocation without waiting for the heap lock!  

### Summary

In this article, we examined how the userland heap allocaor works by debugging into the heap memory with gdb. The discussion is fully based on the `glibc` implementation. The design and behavior of the `glibc` heap allocator are complex but interesting, what we covered here just touches the tip of the iceberg. You can explore more by yourself. 

Moreover, I plan to write a simple version of a heap allocator for learning and teaching purpose. Please keep watching my blog!

