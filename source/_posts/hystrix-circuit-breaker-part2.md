---
title: "Circuit breaker and Hystrix: part two - timeout"
date: 2021-05-30 16:41:20
tags:
---

### Background

In the second article of this series, I will review the source code of `hystrix-go` project to understand how to design a `circuit breaker` and how to implement it with Golang. 

If you're not familiar with `circuit breaker` pattern or `hystrix-go` project, please check my previous [article](https://baoqger.github.io/2021/05/21/hystric-circuit-breaker-part1/) about it.  


### Three service degradation strategies

`Hystrix` provides three different service degradation strategies to avoid the `cascading failure` happening in the entire system: `timeout`, `maximum concurrent request numbers` and `request error rate`. 

- **timeout**: if the service call doesn't return response successfully within a predefined time duration, then the fallback logic will run. This strategy is the simplest one. 
- **maximum concurrent request numbers**: when the number of concurrent requests is beyond the threshold, then the fallback logic will handle the following request. 
- **request error rate**: `hystrix` will record the response status of each service call, after the error rate reaches the threshold, the breaker will be open, and the fallback logic will execute before the breaker status changes back to closed. `error rate` strategy is the most complex one. 

This can be seen from the basic usage of `hystrix` as follows:

```golang
import (
    "github.com/afex/hystrix-go/hystrix"
    "time"
)

hystrix.ConfigureCommand("my_command", hystrix.CommandConfig{
	Timeout:               int(10 * time.Second),
	MaxConcurrentRequests: 100,
	ErrorPercentThreshold: 25,
})

hystrix.Go("my_command", func() error {
	// talk to dependency services
	return nil
}, func(err error) error {
	// fallback logic when services are down
	return nil
})
```

In the above usage case, you can see that `timeout` is set to 10 seconds, the maximum request number is 100, and the error rate threshold is 25 percentages.

In the consumer application level, that's nearly all of the configuration you need to setup. `hystrix` will make the magin happen internally. 

In this series of articles, I plan to show you the internals of `hystrix` by reviewing the source code. 

Let's start from the easy ones: `timeout` and `max concurrent requests`, in this article. In the next articles, I'll introduce `request error rate`. 

### Timeout and Max concurrent requests

Based on the above example, you can see `Go` function is the door to the source code of `hystrix`, so let's start from it as follows: 

```go
func Go(name string, run runFunc, fallback fallbackFunc) chan error {
	runC := func(ctx context.Context) error {
		return run()
	}
	var fallbackC fallbackFuncC
	if fallback != nil {
		fallbackC = func(ctx context.Context, err error) error {
			return fallback(err)
		}
	}
	return GoC(context.Background(), name, runC, fallbackC)
}
```
`Go` function accept three parameters: 
- **name**: the command name, which is bound to the `circuit` created inside hystrix. 
- **run**: a function contains the normal logic which send request to the dependency service.
- **fallback**: a function contains the fallback logic.

`Go` function just wraps `run` and `fallback` with `Context`, which is used to control and cancel goroutine, if you're not familiar with it then refer to my previous [article](https://baoqger.github.io/2021/04/26/golang-context-source-code/). Finally it will call `GoC` function.

`GoC` function goes as follows: 

```go
func GoC(ctx context.Context, name string, run runFuncC, fallback fallbackFuncC) chan error {
	// construct a new command instance
	cmd := &command{
		run:      run,
		fallback: fallback,
		start:    time.Now(),
		errChan:  make(chan error, 1),
		finished: make(chan bool, 1),
	}
	// get circuit by command name
	circuit, _, err := GetCircuit(name)
	if err != nil {
		cmd.errChan <- err
		return cmd.errChan
	}
	cmd.circuit = circuit
	//declare a condition variable sync.Cond: ticketCond, to synchronize among goroutines
	//declare a flag variable: ticketChecked, work together with ticketCond
	ticketCond := sync.NewCond(cmd)
	ticketChecked := false
	// declare a function: returnTicket, will execute when a concurrent request is done to return `ticket`
	returnTicket := func() {
		cmd.Lock()
		for !ticketChecked {
			ticketCond.Wait()
		}
		cmd.circuit.executorPool.Return(cmd.ticket)
		cmd.Unlock()
	}
	// declare a sync.Once instance: returnOnce, make sure the returnTicket function execute only once
	returnOnce := &sync.Once{}

	// declare another function: reportAllEvent, used to collect the metrics
	reportAllEvent := func() {
		err := cmd.circuit.ReportEvent(cmd.events, cmd.start, cmd.runDuration)
		if err != nil {
			log.Printf(err.Error())
		}
	}
	// launch a goroutine which executes the `run` logic
	go func() {
		defer func() { cmd.finished <- true }()

		if !cmd.circuit.AllowRequest() {
			cmd.Lock()
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrCircuitOpen)
				reportAllEvent()
			})
			return
		}

		cmd.Lock()
		select {
		case cmd.ticket = <-circuit.executorPool.Tickets:
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
		default:
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrMaxConcurrency)
				reportAllEvent()
			})
			return
		}

		runStart := time.Now()
		runErr := run(ctx)
		returnOnce.Do(func() {
			defer reportAllEvent()
			cmd.runDuration = time.Since(runStart)
			returnTicket()
			if runErr != nil {
				cmd.errorWithFallback(ctx, runErr)
				return
			}
			cmd.reportEvent("success")
		})
	}()
	// launch the second goroutine for timeout strategy
	go func() {
		timer := time.NewTimer(getSettings(name).Timeout)
		defer timer.Stop()

		select {
		case <-cmd.finished:
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

	return cmd.errChan
}
```
I admit it's complex, but it's also the core of the entire `hystrix` project. Be patient, let's review it bit by bit carefully. 

outline

Go => GoC => Circuit => executorPool, tickets

sync.Once, sync.Cond simply explain

reportEvent leave to next post

fallback logic
