---
title: Golang Context package source code analysis
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

It's just interface, it's very hard to image how to use it. So let's continue reviewing some entities implement such interface. 

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
You can see that `emptyCtx` first declared as a new customized type based on `int`.  In fact, it's not important  that `emptyCtx` is based on `int`, `string` or whatever. The important thing is all the four methods defined in interface `Context` return `nil`. So the root context **is never canceled, has no values, and has no deadline**. 