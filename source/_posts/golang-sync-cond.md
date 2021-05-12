---
title: " Golang synchronization primitives source code analysis: part one - sync.Once"
date: 2021-05-11 20:07:55
tags:
---

### Background

In the following series of posts, I will take a in-depth look at the `synchronization primitives` provided by Golang. 

Although the recommended synchronization mechanism in Golang is `channel`, there are several powerful `synchronization primitives` provided in Golang `sync` pakcage. Based on the official document, **Other than the Once and WaitGroup types, most are intended for use by low-level library routines**. If you read the code of low level open source projects or the standard packages, you will see the use of synchronization primitives in `sync` package a lot. 

As the first post in this series, let's check the source code of `sync.Once` which is also the simplest one.

### sync.Once

If you have several logics running in various go-routines, and you want only one of the logics will execute finally. For this kind of scenario, `sync.Once` is a perfact option for you. 

Let's review the source code of `sync.Once`, which is inside the `once.go` file: 

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
Struct `Once` has a status flag `done` whose value is `0` when it is initialized. The logic you want to execute is passed as a paremeter to the `Do()` method. When `Do` is called for the first time, the logic will execute and then `done` flag is set to `1`. As a result, other calls to `Do` will do nothing. 




