---
title: "Why build heap is a linear operation"
date: 2018-10-08 07:43:52
tags: priority queue, data structure, algorithm, time complexity, Big O notation
---

### Background

In this article, I will focus on the topic of `data structure and algorithms` (in my eyes, one of the most important skills for software engineers). Someday I came across one [question](https://stackoverflow.com/questions/9755721/how-can-building-a-heap-be-on-time-complexity) goes like this: *how can building a heap be O(n) time complexity?* This question confuses me for a while, so I did some investigation and research on it. This article will share what I learned during this process, which covers the following points:
- What is a heap data structure? How does a heap behave? 
- How to implement a completed heap in C programming?
- How to do the time complexity analysis on building the heap?

### Basics of Heap

Before we dive into the implementation and time complexity analysis, let's first understand the `heap`. 

As a data structure, the `heap` was created for the heapsort sorting algorithm long ago. Besides heapsort, `heaps` are used in many famous algorithms such as [Dijkstra’s algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) for finding the shortest path. Essentially, *heaps are the data structure you want to use when you want to be able to access the maximum or minimum element very quickly.*

In computer science, a `heap` is a *specialized tree-based* data structure. A common implementation of a `heap` is the *binary heap*, in which the tree is a *binary tree*. 

So a `heap` can be defined as a binary tree, but with two additional properties (that's why we said it is a specialized tree):

- **shape property**: a binary heap is a *complete* binary tree. So what is a *complete* binary tree? That is all levels of the tree, except possibly the last one(deepest) are fully filled, and, if the last level of the tree is not complete, the nodes of that level are filled from left to right. The complete binary tree is one type of binary tree, in detail,  you can refer to this [document](https://en.wikipedia.org/wiki/Binary_tree#Types_of_binary_trees) to learn more about it.

- **Heap property**: the key stored in each node is either greater than or equal to (max-heaps) or less than or equal to (min-heaps) the keys in the node's children. 

The following image shows a binary max-heap based on tree representation: 

<img src="/images/heap.png" title="heap" width="400px" height="300px">

The `heap` is a powerful data structure; because you can insert an element and extract(remove) the smallest or largest element from a min-heap or max-heap with only **O(log N)** time. That's why we said that if you want to access to the maximum or minimum element very quickly, you should turn to `heaps`. In the next section, I will examine how heaps work by implementing one in C programming. 

Note: The heap is closely related to another data structure called the [`priority queue`](https://en.wikipedia.org/wiki/Priority_queue).  The `priority queue` can be implemented in various ways, but the `heap` is one maximally efficient implementation and in fact, priority queues are often referred as "heaps", regardless of how they may be implemented. 

### Implementation of Heap

Because of the `shape property` of heaps, we usually implement it as an array, as follows:
- Each element in the array represents a node of the heap.
- The parent/child relationship can be defined by the elements' indices in the array. Given a node at index `i`, the left child is at index `2*i + 1` and the right child is at index `2*i + 2`, and its parent is at index `⌊(i-1)/2⌋` (`⌊⌋` means Floor operation). 

### How heapify works 

Both the insert and remove operations modify the heap to conform to the shape property first, by adding or removing from the end of the heap. Then the heap property is restored by traversing up or down the heap. Both operations take O(log n) time.

```
// Perform a down-heap or heapify-down operation for a max-heap
// A: an array representing the heap, indexed starting at 1
// i: the index to start at when heapifying down
Max-Heapify(A, i):
    left ← 2×i
    right ← 2×i + 1
    largest ← i
    
    if left ≤ length(A) and A[left] > A[largest] then:
        largest ← left

    if right ≤ length(A) and A[right] > A[largest] then:
        largest ← right
    
    if largest ≠ i then:
        swap A[i] and A[largest]
        Max-Heapify(A, largest)
```

```
// Perform a down-heap or heapify-down operation for a min-heap
// A: an array representing the heap, indexed starting at 1
// i: the index to start at when heapifying down
Max-Heapify(A, i):
    left ← 2×i
    right ← 2×i + 1
    smallest ← i
    
    if left ≤ length(A) and A[left] < A[smallest] then:
        smallest ← left

    if right ≤ length(A) and A[right] < A[largest] then:
        smallest ← right
    
    if smallest ≠ i then:
        swap A[i] and A[smallest]
        Max-Heapify(A, smallest)
```

heapify-up goes as follows: 

```
// Perform a down-heap or heapify-down operation for a max-heap
// A: an array representing the heap, indexed starting at 1
// i: the index to start at when heapifying down
Max-Heapify(A, i):
    parent ← (i - 1) / 2
    
    if parent >= 0 and A[parent] < A[i] then:
        swap A[i] and A[parent]
        Max-Heapify(A, parent)
```


recursion的四个基本条件：
1. base case
2. make progress
3. assume all recursions work. Don't do bookkeeping work by yourself.  
4. NO duplicated work for sub problem.

上面的做法都没有base case
### Time complexity for building a heap
