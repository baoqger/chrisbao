---
title: " Golang synchronization primitives source code analysis: part one - sync.Once"
date: 2021-05-11 20:07:55
tags:
---

### Background

In the following series of posts, I will take an in-depth look at the `synchronization primitives` provided by Golang. 

Although the recommended synchronization mechanism in Golang is `channel`, there are several powerful `synchronization primitives` provided in Golang `sync` package. Based on the official document, **Other than the Once and WaitGroup types, most are intended for use by low-level library routines**. If you read the code of low-level open source projects or the standard packages, you will see synchronization primitives in `sync` package frequently. 

As the first post in this series, let's check the source code of `sync.Once`, which is also the simplest one.

### sync.Once

If you have several logics running in various go-routines, and you want only one of the logics will execute finally. For this kind of scenario, `sync.Once` is a perfect option for you. 

Let's review the source code of `sync.Once` defined inside the `once.go` file: 

```golang
package sync

import (
	"sync/atomic"
)

type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```
Struct `Once` has a status flag `done` whose value is `0` when initialized. Wrap the logic you want to execute in a function `f`, and pass this function `f` to the `Do()` method. When `Do` is called for the first time, the logic in `f` executes after that `done` flag is set to `1`, other calls to `Do` don't execute `f`. 

One point may seem misleading is that **If `once.Do(f)` is called multiple times, only the first call will invoke `f`, even if `f` has a different value in each invocation**. Check the following example:

```golang
package main

import (
	"fmt"
	"sync"
	"time"
)

var doOnce sync.Once

func main() {
	go TaskOne()
	go TaskTwo()
	time.Sleep(time.Second)
}

func PrintTask(id int) {
	fmt.Printf("Task %d, Run once\n", id)
}

func TaskOne() {
	doOnce.Do(func() {
		PrintTask(1)
	})
	fmt.Println("Run this every time")
}

func TaskTwo() {
	doOnce.Do(func() {
		PrintTask(2)
	})
	fmt.Println("Run this every time")
}
```
Even `Do` is called twice with different `f` logic, but only the first call is invoked since they are bound to the same instance of `Once`. 

### fast path and slow path

As you saw above, the implementation of `sync.Once` is very not complex. But one question comes to my mind when I double check the code. Why do we need split the logics into two functions `Do` and `doSlow`? Why the second function name is `doSlow`, and what does `slow` mean here?

Do you have similar questions? 

I found the answer in the comments of `once.go` file. 

```golang 
if atomic.LoadUint32(&o.done) == 0 {
	// Outlined slow-path to allow inlining of the fast-path.
	o.doSlow(f)
}
```

Note that it mentioned two words: `fast path` and `slow path`. 

- Fast path is a term used in computer science to describe a path with shorter instruction path length through a program compared to the 'normal' path. For a fast path to be effective it must handle the most commonly occurring tasks more efficiently than the 'normal' path, leaving the latter to handle uncommon cases, corner cases, error handling, and other anomalies. **Fast paths are a form of optimization**. 

In the `Once` case, since the first call to `Do` function will set `done` to 1, so the most common case or status for `Once` is the `done` flag equals to 1. The `fast path` in `Do` method is just for this common case. While the `done` flag equals to initial status 0 can be regarded as uncommon case, which is specially handled in the `doSlow` function. The performance can be optimized in this way.

### hot path

Another very interesting concept worth mentioning is `hot path`, and it occurs in the `Once` struct design.

```golang
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/x86),
	// and fewer instructions (to calculate offset) on other architectures.
	done uint32
	m    Mutex
}
```
At first glance, it's just plain and ordinary struct, but the comments emphasize that `done` **is first in the struct because it is used in the hot path**. It means that `done` is defined as the first field in `Once` struct on purpose. And the purpose is also described in the comment **Placing done first allows more compact instructions on some architectures (amd64/x86), and fewer instructions (to calculate offset) on other architectures.**

What does that mean? I found a great answer in this [post](https://stackoverflow.com/questions/59174176/what-does-hot-path-mean-in-the-context-of-sync-once). The conclusion is:

- A `hot path` is a sequence of instructions executed very frequently.

- When accessing the first field of a structure, we can directly dereference the pointer to the structure to access the first field. To access other fields, we need to provide an `offset` from the first value in addition to the struct pointer.

- In machine code, this offset is an additional value to pass with the instruction which makes it longer. The performance impact is that the CPU must perform an addition of the offset to the struct pointer to get the address of the value to access.

- Thus machine code to `access the first field of a struct is more compact and faster`.

Simply speaking, accessing the first field of a struct is faster since the CPU doesn't need to compute the memory offset!

This is really a good lesson to show the high-level code you programmed can have such a big difference in the bottom level.

















