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

### Function of GoC

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

First of all, the code structure of `GoC` function is as follows:

![GoC](/images/GoC-hystrix.png)


  1. Construct a new `Command` object, which contains all the information for each call to `GoC` function.
  2. Get the `circuit breaker` by name (create it if it doesn't exist) by calling `GetCircuit(name)` function.
  3. Declare condition variable **ticketCond** and **ticketChecked** with `sync.Cond` which is used to communicate between goroutines. 
  4. Declare function **returnTicket**. What is a **ticket**? What does it mean by **returnTicket**? Let's discuss it in detail later.
  5. Declare another function **reportAllEvent**. The same as above, let's discuss it in the following part. 
  6. Declare an instance of `sync.Once`, which is another interesting `synchronization primitives` provided by golang.
  7. Launch two goroutines, each of which contains many logics too. 
  8. Return a `channel` type value

Let's review each of them one by one. 

### command

`command` struct goes as follows, which embeds **sync.Mutex** and defines several fields: 

```golang
type command struct {
	sync.Mutex

	ticket      *struct{}
	start       time.Time
	errChan     chan error
	finished    chan bool
	circuit     *CircuitBreaker
	run         runFuncC
	fallback    fallbackFuncC
	runDuration time.Duration
	events      []string
}
```
Note that `command` object iteself doesn't contain command name information, and its lifecycle is just inside the scope of one `GoC` call. It means that the statistic metrics about the service request like `error rate` and `concurrent request number` are not stored inside command object. Instead, such metrics are stored inside **circuit** field which is `CircuitBreaker` type. 

### CircuitBreaker

As we mentioned in the workflow of `GoC` function, `GetCircuit(name)` is called to get or create the `circuit breaker`. It is implemented inside `circuit.go` file as follows:

```golang
func init() {
	circuitBreakersMutex = &sync.RWMutex{}
	circuitBreakers = make(map[string]*CircuitBreaker)
}

func GetCircuit(name string) (*CircuitBreaker, bool, error) {
	circuitBreakersMutex.RLock()
	_, ok := circuitBreakers[name]
	if !ok {
		circuitBreakersMutex.RUnlock()
		circuitBreakersMutex.Lock()
		defer circuitBreakersMutex.Unlock()

		if cb, ok := circuitBreakers[name]; ok {
			return cb, false, nil
		}
		circuitBreakers[name] = newCircuitBreaker(name)
	} else {
		defer circuitBreakersMutex.RUnlock()
	}

	return circuitBreakers[name], !ok, nil
}
```
The logic is very straightforward. All the circuit breakers are stored in a map object **circuitBreakers** with the **command name** as the key. 

The `newCircuitBreaker` constructor function and `CircuitBreaker` struct are as follows: 

```go
type CircuitBreaker struct {
	Name                   string
	open                   bool
	forceOpen              bool
	mutex                  *sync.RWMutex
	openedOrLastTestedTime int64

	executorPool *executorPool   // used in the strategy of max concurrent request number 
	metrics      *metricExchange // used in the strategy of request error rate
}

func newCircuitBreaker(name string) *CircuitBreaker {
	c := &CircuitBreaker{}
	c.Name = name
	c.metrics = newMetricExchange(name)
	c.executorPool = newExecutorPool(name)
	c.mutex = &sync.RWMutex{}

	return c
}
```

All the fields of `CircuitBreaker` are important to understand how breaker works.

<img src="/images/circuitbreakstruct.png" title="circuitbreak" width="300px" height="400px">

There are two fileds that are not simple type need more analysis, include `executorPool` and `metrics`. 
- **executorPool**: used for `max concurrent request number` strategy which is just this article's topic.
- **metrics**: used for `request error rate` strategy which will be discussed in next article, all right? 

### executorPool

<img src="/images/hystrix-concurrent-architecture.png" title="circuitbreak" width="800px" height="400px">

<img src="/images/metrics_architecture.png" title="circuitbreak" width="800px" height="400px">
We can find `executorPool` logics inside the `pool.go` file:

```go
type executorPool struct {
	Name    string
	Metrics *poolMetrics
	Max     int
	Tickets chan *struct{}
}

func newExecutorPool(name string) *executorPool {
	p := &executorPool{}
	p.Name = name
	p.Metrics = newPoolMetrics(name)
	p.Max = getSettings(name).MaxConcurrentRequests

	p.Tickets = make(chan *struct{}, p.Max)
	for i := 0; i < p.Max; i++ {
		p.Tickets <- &struct{}{}
	}

	return p
}
```

outline

Go => GoC => Circuit => executorPool, tickets

sync.Once, sync.Cond simply explain

reportEvent leave to next post

fallback logic
