---
title: "Golang Context package source code analysis: part 1"
date: 2021-04-26 20:44:48
tags:
---

### Background

As a Golang user and learner, I always think Golang's standard packages are great learning resource, which can provide best practise for both the languange iteself and various software or programming concepts. 

In this post, I will share what I learned about package `context`. 

`context` is widly used in Golang ecosystem, I bet you must often come across it, in fact many standard packages rely on it. 

There are many good [articles](https://golangbyexample.com/using-context-in-golang-complete-guide/) online explaining the background and usage examples of `context`, I won't spend too much time on that, just place a little bit introduction here. 

The problems `context` plans to solve are：

 - Let’s say that you started a function and you need to pass some common parameters to the downstream functions. You cannot pass these common parameters each as an argument to all the downstream functions.
 - You started a goroutine which in turn start more goroutines and so on. Suppose the task that you were doing is no longer needed. Then how to inform all child goroutines to gracefully exit so that resources can be freed up
 - A task should be finished within a specified timeout of say 2 seconds. If not it should gracefully exit or return.
 - A task should be finished within a deadline eg it should end before 5 pm . If not finished then it should gracefully exit and return

You can refer to this [slide](https://talks.golang.org/2014/gotham-context.slide#1) from the author of context package to understand more about the background. 

In this post, I will show you the details of context source code. You can find all the related source code inside `context.go` file. You will notice that `context` package is not big, there are totally 500 lines of codes. Moreover there are many comments, the actual code is only half. So these 200+ lines of code is a great piece of learning resource in my eyes. 

### Source code analysis

#### Context interface and emptyCtx

The most basic data structure of context is the `Context` interface as below:

```golang
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

It's just interface, it's very hard to imagine how to use it. So let's continue reviewing some entities implement such interface. 

When context is used, generally speaking, the first step is creating the root context with `context.Background()` function(the contexts chained together one by one and form a tree structure, and the root context is the first one in the chain). Let's check what it is: 

```golang
var background = new(emptyCtx)

func Background() Context {
	return background
}
```
`Background` function return the `background` which is a global variable declared as `new(emptyCtx)`. So what is `emptyCtx`, let continue:

```golang
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}
```
You can see that `emptyCtx` is declared as a new customized type based on `int`.  In fact, it's not important  that `emptyCtx` is based on `int`, `string` or whatever. The important thing is all the four methods defined in interface `Context` return `nil`. So the root context **is never canceled, has no values, and has no deadline**. 

Let's continue to review other data types.


#### valueCtx and WithValue

As mentioned above, one typical usage of context is passing data. In this case, you need to create a `valueCtx` with `WithValue` function. For example, the following example:

```golang
rootCtx := context.Background()

childCtx := context.WithValue(rootCtx, "msgId", "someMsgId")
```

`WithValue` is a function has only one return value:

```golang
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

Please ignore the `reflectlite` part, I will give a in-depth discussion about it in another post. In this post, we only need to care the return value type is `&valueCtx`:

```golang
type valueCtx struct {
	Context
	key, val interface{}
}
```
There is one interesting Golang language feature here: `embedding`, which realizes `inheritance`. In this case, `valueCtx` has all the four methods defined in `Context`.
In fact, `embedding` is worthy much more discussion. Simplying speaking, there are 3 types of embedding: **struct in struct**, **interface in interface** and **interface in struct**. `valueCtx` is the last type, you can refer to this great [post](https://eli.thegreenplace.net/2020/embedding-in-go-part-1-structs-in-structs/)

When you want to get the value out, you can use the `Value` method: 

```golang
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

If the provided `key` parameter doesn't match the current context's key, then the parent context's `Value` method will be called. If we still can't find the key, the parent context's will call its parent as well. The search will pass along the chain until the root node which will return `nil` as we mentioned above:

```golang
func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

Next, let's review another interesting type: `cancelCtx`

#### cancelCtx and WithCancel

First, let's see how to use `cancelCtx` and `WithCanel` with a simple example:

```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	cancelCtx, cancelFunc := context.WithCancel(context.Background())
	go task(cancelCtx)
	time.Sleep(time.Second * 3)
	cancelFunc()
	time.Sleep(time.Second * 3)
}

func task(ctx context.Context) {
	i := 1
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Err())
			return
		default:
			fmt.Println(i)
			time.Sleep(time.Second * 1)
			i++
		}
	}
}
```
When main goroutine wants to cancel `task` goroutine, it can just call `cancelFunc`. Then the task goroutine will exit and stop running. In this way, goroutine management will be easy task. Let's review the code:

```golang
type CancelFunc func()

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
`cancelCtx` is complex, let's go through bit by bit. 

`WithCancel` returns two values, the first one `&c` is type `cancelCtx` which is created with `newCancelCtx`, the second one `func() { c.cancel(true, Canceled) }` is type `CancenlFunc`(just a general function). 

Let's review `cancelCtx` firstly:

```golang
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

`Context` is embedded inside `cancelCtx` as well. Also it defines several other fields. Let's see how it works by checking the receiver methods:

```golang
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```

`Done` method returns channel `done`. In the above demo, **task** goroutine listen for cancel signal from this done channel like this:
```golang
select {
case <-ctx.Done():
	fmt.Println(ctx.Err())
	return
...
```
The signal is trigger by calling the cancle function, so let's review what happens inside it and how the signals are sent to the channel. All the logic is inside `cancel` method of `cancelCtx`:

```golang
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	// set the err property when cancel is called for the first time
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
As shown above, `cancelCtx` has four properties, we can understand their purpose clearly in this `cancel`: 
 - `mu`: a general lock to make sure goroutine safe and avoid race condition;
 - `err`: a flag representing whether the cancelCtx is cancelled or not. When the cancelCtx is created, `err` value is `nil`. When `cancel` is called for the first time, it will be set by `c.err = err`;
 - `done`: a channel which sends cancel signal. To realize this, context just `close` the done channel instead of send data into it. This is an **interesting point** which is different from my initial imagination before I review the source code. Yes, after a channel is closed, the receiver can still get `zero value` from the closed channel based on the channel type. Context just make use of this feature.
 - `children`: a `Map` containing all its child contexts. When current context is cancelled, the cancel action will be propogated to the children by calling `child.cancel(false, err)` in the for loop. Then next question is when the parent-child relationship is established? The secret is inside the `propagateCancel()` function;

```golang
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```
`propagateCancel` contains many logics, and some of them can't be understood easily, I will write another post for those parts. But in this post, we only need to understand how to establish the relationship between parent and child for genernal cases. 

The key point is function `parentCancelCtx`, which is used to find the innermost cancellable ancestor context:

```golang
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	// Value() will propagate to the root context
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	ok = p.done == done
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```
You can notice that `Value` method is called, since we analyzed in the above section, `Value` will pass the search until the root context. Great. 

Back to the `propagateCancel` function, if cancellable ancestor context is found, then current context is added into the `children` hash map as below:

```golang
if p.children == nil {
	p.children = make(map[canceler]struct{})
}
p.children[child] = struct{}{}
```
The relationship is established. 

### Summary

In this article, we review the source code of `Context` package and understand how `Context`,  `valueCtx` and `cancelCtx` works. 

`Context` contains the other two types of context: `timeOut` context and `deadLine` context, Let's work on that in the second part of this post series. 




