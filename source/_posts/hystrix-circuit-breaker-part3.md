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

### sync.Cond

### Fallback 



