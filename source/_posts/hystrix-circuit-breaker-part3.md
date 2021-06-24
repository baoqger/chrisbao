---
title: "Circuit breaker and Hystrix: part three - timeout"
date: 2021-06-18 15:54:02
tags:
---

In the [previous article](https://baoqger.github.io/2021/05/30/hystrix-circuit-breaker-part2/), we reviewed the `max concurrent request number` service degradation strategy. But some detailed techniques are not explained very clearly, which will be talked about in this article. And we will explore `timeout` strategy as well.

### Timeout

Compared with `max concurrent request number` strategy, `timeout` is very straightforward to understand. 

As we mentioned in the previous article, the core logic of `hystrix` is inside the `GoC` function. `GoC` function internally run two goroutines. You already see that the first goroutine contains the logic to send request to the target service and the strategy of `max concurrent request number`. How about the second goroutine? Let's review it as follows:

```golang
	go func() {
		timer := time.NewTimer(getSettings(name).Timeout)
		defer timer.Stop()

		select {
		case <-cmd.finished:
			// returnOnce has been executed in another goroutine
		case <-ctx.Done():
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ctx.Err())
				reportAllEvent()
			})
			return
		case <-timer.C:
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrTimeout)
				reportAllEvent()
			})
			return
		}
	}()
```

Note that A **Timer** is created with the timeout duration value from the settings. And a `select` statement lets this goroutine wait until one `case` condition receive value from the channel. The **timeout** case is just the 3nd one, which will run fallback logic with **ErrTimeout** error message. 

So far you should be clear about the main structure and functionalities of these two goroutines. But in detail, there are two Golang techniques need your attention: `sync.Once` and `sync.Cond`.  

### sync.Once

I may already notice the following code block, which is used several times inside `GoC` function. 

```golang
returnOnce.Do(func() {
	returnTicket()
	cmd.errorWithFallback(ctx, ErrTimeout) // with various error types 
	reportAllEvent()
})
```

**returnOnce** is type of `sync.Once`, which makes sure that the callback function of `Do` method only run once among different goroutines. 

In this specific case, it can make sure that both **returnTicket()** and **reportAllEvent()** execute only once. This really makes sense, because if **returnTicket()** run multiple times for one `GoC` call, then the current concurent request number will not be correct, right? 

Previously, I wrote another article about `sync.Once` in detail, you can refer to [that article](https://baoqger.github.io/2021/05/11/golang-sync-once/) for more in-depth explanation. 

### sync.Cond

The implementation of **returnTicket** function goes as follows:

```golang
ticketCond := sync.NewCond(cmd)
ticketChecked := false
returnTicket := func() {
	cmd.Lock()
	for !ticketChecked {
		ticketCond.Wait() // hang the current goroutine
	}
	cmd.circuit.executorPool.Return(cmd.ticket)
	cmd.Unlock()
}
```
**ticketCond** is a condition variable, and in Golang world it is type of `sync.Cond`. 

Condition variable is useful in communication between different goroutines. Concretely, `Wait` method of `sync.Cond`will hung the current goroutine, and `Signal` method will wake up the blocking goroutine to continue executing. 

In `hystrix` case, when **ticketChecked** is **false**, which means the current `GoC` call is not finished and the **ticket** should not be returned yet. So **ticketCond.Wait()** is called to block this goroutine and wait until the `GoC` call is completed which is notified by `Signal` method. 

```golang
ticketChecked = true
ticketCond.Signal()
```

Note that the above two lines of code are always called together. **ticketChecked** is set to **true** means that the current `GoC` call is finished and the **ticket** is ready to return. Moreover, the `Wait` method to hang the goroutine is placed inside a **for** loop, which is also a best practise technique. 

For more explanation about `sync.Cond`, please refer to my [another article](https://baoqger.github.io/2021/05/14/golang-sync-cond/).

### Fallback 



