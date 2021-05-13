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

  











