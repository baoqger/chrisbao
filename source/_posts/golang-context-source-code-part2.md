---
title: "Golang Context package source code analysis: part 2"
date: 2021-04-28 13:38:51
tags:
---

### Background

In the [last post](https://baoqger.github.io/2021/04/26/golang-context-source-code/), I shared the first part about the `context` package: `valueCtx` and `cancelCtx`. Let us continue the journey to discover more in this post. 


### WithTimeout and WithDeadline 

As usual, let us start with an example: 
```golang
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    cancelCtx, cancel := context.WithTimeout(context.Background(), time.Second*3)
    defer cancel()
    go task(cancelCtx)
    time.Sleep(time.Second * 4)
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
Since we already know the behavior of `cancelCtx`, it's quite straightforward to understand how `WithTimeout` works. It accepts a timeout duration after which the `done` channel will be closed and context will be canceled. And a cancel function will be returned as well, which can be called in case the context needs to be canceled before timeout. 

`WithDeadline` usage is quite similar to `WithTimeout`, you can find related example easily. Let us review the source code:

```golang
type timerCtx struct {
	cancelCtx
	timer *time.Timer 
	deadline time.Time
}

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
Since `WithTimeout` and `WithDeadline` have many common points between them, so they share the same type of context: `timerCtx`, which embeds `cancelCtx` and defines two more properties: `timer` and `deadline`. 

Let us review what happens when we create a `timerCtx`:
```golang
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	// Get deadline time of parent context. 
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil { // 'err' field of the embedded cancelCtx is promoted 
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
Compared to `WithCancle` and `WithValue`, `WithDeadline` is more complex, let us go through bit by bit.

Firstly, `parent.Deadline` will get the deadline time for parent context. The `Deadline` method signature was defined in the `Context` interface as below:

```golang
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	...
}
```

In the context package, only `emptyCtx` and `timerCtx` type implement this method:

```golang
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```

So when we call `parent.Deadline()`, if the parent context is also type of `timerCtx` which implements its own `Deadline()` method, then you can get the deadline time of the parent context. Otherwise if the parent context is type of `cancelCtx` or `valueCtx`, then finally the `Deadline()` method of `emptyCtx` will be called and you will get the zero value of type `time.Time` and `bool` (if you have interest, you can verify by yourself the zero value: **0001-01-01 00:00:00 +0000 UTC** and **false**).  

If parent's deadline is earlier than the passed in deadline parameter, then directly return a `cancelCtx` by calling `WithCancel(parent)`. Of course when the passed in deadline is reasonable, we need to create a `timerCtx`: 

```golang
//inside WithDeadline() function
...
c := &timerCtx{
	cancelCtx: newCancelCtx(parent),
	deadline:  d,
}
propagateCancel(parent, c)
...
```
In the above code, you see `propagateCancel` method again, I have discussed about it in the last post, if you don't understand it, please refer [here](https://baoqger.github.io/2021/04/26/golang-context-source-code/).

Similar to `cancelCtx`, `timerCtx` sends the context cancel signal by closing the done channel by calling its own `cancel` method. There two scenarios when cancelling the context:
 - timeout cancel: when the deadline exceeded, automatically close the done channel; 

```golang
// inside WithDeadline function
...
// timeout cancel
c.timer = time.AfterFunc(dur, func() {
	c.cancel(true, DeadlineExceeded)
})
...
```
 - manual cancel: call the returned cancel function to close the done channel before the deadline;
```golang
// inside WithDeadline function
...
// return the cancel function as the second return value
return c, func() { c.cancel(true, Canceled) }
...
```

Both scenarios call `cancel` method: 

```golang
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err) // close the done channel and set err field
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		// Note: timerCtx c's parent is c.cancelCtx.Context
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	// stop and clean the timer
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
`timerCtx` implements `cancel` method to stop and reset the timer then delegate to `cancelCtx.cancel`. 

### Summary

In the second part of this post series, we discussed how `timeout` and `deadline` context are implemented in the source code level. In this part, Golang struct embedding technique is used a lot, you can compare it with traditional OOP solution to have a deep understanding.





