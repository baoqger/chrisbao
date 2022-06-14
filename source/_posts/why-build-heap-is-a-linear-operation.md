---
title: "How to build a Heap in linear time complexity"
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

<img src="/images/array-representation.png" title="heap" width="400px" height="300px">

Based on the above model, let's start implementing our heap. As we mentioned, there are two types of heaps: min-heap and max-heap, in this article, I will work on `max-heap`. The difference between max-heap and min-heap is trivial, you can try to write out the min-heap after you understand this article. 

The completed code implementation is inside this Github [repo](https://github.com/baoqger/max-heap-in-c). 

First, let's define the interfaces of max-heap in the header file as follows:

```c
// maxheap.h file
#ifndef _MAXHEAP_H
#define _MAXHEAP_H

typedef int key_type;
typedef struct _maxheap *maxheap; // opaque type

maxheap maxheap_create();
maxheap maxheap_heapify(const key_type *array, int n);
void maxheap_destroy(maxheap);

int maxheap_findmax(maxheap);
void maxheap_insert(maxheap, key_type);
void maxheap_deletemax(maxheap);

int maxheap_is_empty(maxheap);
int maxheap_size(maxheap);
void maxheap_clear(maxheap);

#endif
```

We define the max-heap as `struct _maxheap` and hide its implementation in the header file. And expose this struct in the interfaces via a handler(which is a pointer) `maxheap`. This technique in C program is called [`opaque type`](https://stackoverflow.com/questions/2301454/what-defines-an-opaque-type-in-c-and-when-are-they-necessary-and-or-useful). `Opaque type` simulates the encapsulation concept of OOP programming. So that the internal details of a type can change without the code that uses it having to change. The detailed implementation goes as following:

```c
// maxheap.c file
struct _maxheap {
	key_type* array;
	int max_size;
	int cur_size;
};
```

The max-heap elements are stored inside the `array` field. The capacity of the array is defined as field `max_size` and the current number of elements in the array is `cur_size`. 

Next, let's go through the interfaces one by one (most of the interfaces are straightforward, so I will not explain too much about them). The first one is `maxheap_create`, which constructs an instance of `maxheap` by allocating memory for it. 

```c
// maxheap.c file
maxheap maxheap_create() {
	maxheap h = (maxheap)malloc(sizeof(struct _maxheap));
	if (h == NULL) {
		fprintf(stderr, "Not enough memory!\n");
		abort();
	}
	h->max_size = 64;
	h->cur_size = -1;
	h->array = (key_type*) malloc(sizeof(key_type)*(h->max_size));
	if (h->array == NULL) {
		fprintf(stderr, "Not enough memory!\n");
		abort();
	}
	return h;
}
```

The initial capacity of the max-heap is set to 64, we can dynamically enlarge the capacity when more elements need to be inserted into the heap: 

```c
// maxheap.c file
static void maxheap_double_capacity(maxheap h) {
	int new_max_size = 2 * h->max_size;
	key_type* new_array = (key_type*) malloc(sizeof(key_type)* (new_max_size));

	if (new_array == NULL) {
		fprintf(stderr, "Not enough memory!\n");
		abort();
	}
	for(int i = 0; i < h->cur_size; i++) {
		new_array[i] = h->array[i];
	}
	free(h->array);
	h->array = new_array;
	h->max_size = new_max_size;
}
```
This is an internal API, so we define it as a [`static`](https://www.tutorialspoint.com/static-functions-in-c) function, which limits the access scope to its object file. 

When the program doesn't use the max-heap data anymore, we can destroy it as follows:

```c
// maxheap.c file
void maxheap_destroy(maxheap h) {
	assert(h);
	free(h->array);
	free(h);
}
```
Don't forget to release the allocated memory by calling `free`. 

Next, let's work on the difficult but interesting part: insert an element in **O(log N)** time. The solution goes as follows:
- Add the element to the end of the array. (The end of the array corresponds to the leftmost open space of the bottom level of the tree).
- Compare the added element with its parent; if they are in the correct order(parent should be greater or equal to the child in max-heap, right?), stop.
- If not, swap the element with its parent and return to the above step until reaches the top of the tree(the top of the tree corresponds to the first element in the array). 

The first step of adding an element to the array's end conforms to the **shape property** first. Then the **heap property** is restored by traversing up the heap. The recursive traversing up and swapping process is called `heapify-up`. It is can be illustrated by the following pseudo-code:
```
Max-Heapify-Up(A, i):
    parent ← (i - 1) / 2
    
    if parent >= 0 and A[parent] < A[i] then:
        swap A[i] and A[parent]
        Max-Heapify-Up(A, parent)
```
The implementation goes as follows:

```c
// maxheap.c file
static void maxheap_swap(maxheap h, int i, int j) {
	assert(h && i >=0 && i <= h->cur_size && j >= 0 && j <= h->cur_size);
	key_type tmp = h->array[i];
	h->array[i] = h->array[j];
	h->array[j] = tmp;
}

static void maxheap_heapifyup(maxheap h, int k) {
	assert(h && k >= 0 && k <= h->cur_size);

	while(k>=0 && h->array[k] > h->array[k/2]) {
		maxheap_swap(h, k/2, k);
		k /= 2;
	}
}

void maxheap_insert(maxheap h, key_type key) {
	assert(h);

	h->cur_size += 1;
	// make sure there is space
	if (h->cur_size == h->max_size) {
		maxheap_double_capacity(h);
	}

	// add at the end
	h->array[h->cur_size] = key;
	
	// restore the heap property by heapify-up
	maxheap_heapifyup(h, h->cur_size);

}
```
The number of operations requried in `heapify-up` depends on how many levels the new element must rise to satisfy the heap property. So the worst-case time complexity should be the height of the binary heap, which is **log N**. And appending a new element to the end of the array can be done with constant time by using `cur_size` as the index. As a result, the total time complexity of the insert operation should be **O(log N)**. 

Similarly, next, let's work on: extract the root from the heap while retaining the heap property in **O(log N)** time. The solution goes as follows:
- Replace the first element of the array with the element at the end. Then delete the last element. 
- Compare the new root with its children; if they are in the correct order, stop.
- If not, swap the element with its child and repeat the above step.

This similar traversing down and swapping process is called `heapify-down`. `heapify-down` is a little more complex than `heapify-up` since the parent element needs to swap with the *larger* children in the max heap. The implementation goes as follows: 

```c
// maxheap.c file
int maxheap_is_empty(maxheap h) {
	assert(h);
	return h->cur_size < 0;
}

static void maxheap_heapifydown(maxheap h, int k) {
	assert(h);
	
	while(2*k <= h->cur_size) {
		int j = 2*k;
		if (j<h->cur_size && h->array[j+1] > h->array[j]) {
			j++;
		}
		if (h->array[k] >= h->array[j]) {
			break;
		}
		maxheap_swap(h, k, j);
		k = j;
	}
}

int maxheap_findmax(maxheap h) {
	if (maxheap_is_empty(h)) {
		fprintf(stderr, "Heap is empty!\n");
		abort();
	}

	// max is the first position
	return h->array[0];
}

void maxheap_deletemax(maxheap h) {
	if (maxheap_is_empty(h)) {
		fprintf(stderr, "Heap is empty!\n");
		abort();
	}
	// swap the first and last element
	maxheap_swap(h, 0, h->cur_size);
	h->cur_size -= 1;
	
	maxheap_heapifydown(h, 0);
}
```
Based on the analysis of `heapify-up`, similarly, the time complexity of extract is also **O(log n)**.

In the next section, let's go back to the question raised at the beginning of this article. 
### The time complexity of building a heap

What's the time complexity of building a heap? The first answer that comes to my mind is **O(n log n)**. Since the time complexity to insert an element is *O(log n)*, for n elements the insert is repeated n times, so the time complexity is *O(n log n)*. Right?

We can use another optimal solution to build a heap instead of inserting each element repeatedly. It goes as follows:

- Arbitrarily putting the n elements into the array to respect the **shape property**.
- Starting from the lowest level and moving upwards, sift the root of each subtree downward as in the `heapify-down` process until the **heap property** is restored. 

This process can be illustrated with the following image:

<img src="/images/build-heap.png" title="heap" width="400px" height="300px">

This algorithm can be implemented as follows: 

```c
maxheap maxheap_heapify(const key_type* array, int n) {
	assert(array && n > 0);
	
	maxheap h = (maxheap)malloc(sizeof(struct _maxheap));
	if(h == NULL) {
		fprintf(stderr, "Not enough memory!\n");
		abort();
	}
	h->max_size = n;
	h->cur_size = -1;
	h->array = (key_type*)(malloc(sizeof(key_type)*h->max_size));
	if(h->array == NULL) {
		fprintf(stderr, "Not enough memory!\n");
		abort();
	}
	h->cur_size = n-1;
	for (int i=0; i< n; i++) {
		h->array[i] = array[i];
	}
	// small trick here. don't need start the loop from the end
	for (int i = h->max_size/2; i >= 0; i--) { 
		maxheap_heapifydown(h, i);
	}
	return h;
}
```
Next, let's analyze the time complexity of this above process. Suppose there are *n* elements in the heap, and the height of the heap is *h* (for the heap in the above image, the height is 3). Then we should have the following relationship: 

{% katex %}  2^{h} <= n <= 2^{h+1} - 1 {% endkatex %}

When there is only one node in the last level then {% katex %} n = 2^{h} {% endkatex %}. And when the last level of the tree is fully filled then {% katex %} n = 2^{h+1} - 1 {% endkatex %}

And start from the bottom as level *0* (the root node is level *h*), in level *j*, there are at most {% katex %} 2^{h-j} {% endkatex %} nodes. And each node at most takes *j* times swap operation. So in level *i*, the total number of operation is {% katex %} j*2^{h-j} {% endkatex %}.

So the total running time for building the heap is proportional to:

{% katex %} 
T(n) = \displaystyle\sum_{j=0}^h {j*2^{h-j}} = \displaystyle\sum_{j=0}^h {j* \frac {2^{h}} {2^{j}}}
{% endkatex %}

If we factor out the {% katex %} 2^{h} {% endkatex %} term, then we get:

{% katex %} 
T(n) = 2^{h} \displaystyle\sum_{j=0}^h {\frac {j} {2^{j}}}
{% endkatex %}

As we know, {% katex %} \displaystyle\sum_{j=0}^{\infin} {\frac {j} {2^{j}}} {% endkatex %} is a series converges to 2 (in detail, you can refer to this [wiki](https://en.wikipedia.org/wiki/Series_(mathematics))).  

Using this we have:

{% katex %} 
T(n) = 2^{h} \displaystyle\sum_{j=0}^h {\frac {j} {2^{j}}} <= 2^{h} \displaystyle\sum_{j=0}^{\infin} {\frac {j} {2^{j}}} <= 2^{h}*2 = 2^{h + 1}
{% endkatex %}

Based on the condition {% katex %} 2^{h} <= n {% endkatex %}, so we have:

{% katex %} 
T(n) <= 2n = O(n)
{% endkatex %}

Now we prove that building a heap is a linear operation.

### Summary

In this article, we examined what is a `Heap` and understand how it behaves(`heapify-up` and `heapify-down`) by implementing it. More importantly, we analyze the time complexity of building a heap and prove it's a linear operation.




