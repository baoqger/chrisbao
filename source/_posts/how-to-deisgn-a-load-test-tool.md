---
title: "How to design a load performance test CLI tool"
date: 2021-07-04 10:18:43
tags:
---

### Background

When you want to do the load performance test to your HTTP backend service, a handy and powerful tool can make your job mush easier. For example, `ApacheBench` (short for [ab](https://en.wikipedia.org/wiki/ApacheBench)) is widely used in this field. But it is not today's topic. Instead, I want to introduce [Hey](https://github.com/rakyll/hey) written in `Golang` and supports the same functionality as `ab`.  

`Hey` usage goes as follows:

```golang
Usage: hey [options...] <url>

Options:
  -n  Number of requests to run. Default is 200.
  -c  Number of workers to run concurrently. Total number of requests cannot
      be smaller than the concurrency level. Default is 50.
  -q  Rate limit, in queries per second (QPS) per worker. Default is no rate limit.
  -z  Duration of application to send requests. When duration is reached,
      application stops and exits. If duration is specified, n is ignored.
      Examples: -z 10s -z 3m.
  ...
// other options are hidden
```

I didn't list all the options, but just show several ones related to this article's content. As you can see in the above list, `Hey` can support different practical features, such as **multiple workers** to run in the **concurrent** style and **rate limit** by **queries per second (QPS)**. It can also support **run by duration** and **run by request number** two modes.

In this article, we can review the design and implementation of `Hey` to see how to make a load performance testing tool.


### Architecture Design
The design of `Hey` is not complex and the design can be divided into the following three parts:
- Control logic: the main workflow like how to set up multiple concurrent workers, how to control QPS rate limiter and how to exit the process when duration is reached; 
- HTTP request configuration: the headers or parameters configuration needed to send  request;   
- Test report: print or save the result after the load testing finish. 

This article will focus on the first item (since it is the real interesting part) to show how to use `Golang`'s concurrent programming techniques to realize these features.

### Exit the process

In the `hey.go` file, you can find the entry point **main** function. Let's hide the boilerplate code and review  the core logic in the **main** function as follows:

```golang
    w := &requester.Work{
		N:  num,  // number of request
		C:  conc, // number of concurrent works
		QPS: q,   // QPS setting 
		results  chan *result, // channel for request response
		stopCh   chan struct{}, // channle for stop the worker 		    
        //  hide the other fields
    }
	w.Init()

	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	go func() {
		<-c
		w.Stop()
	}()
    // if the duration is set, then launch another goroutine
	if dur > 0 { 
		go func() {
			time.Sleep(dur)
			w.Stop()
		}()
	}
	w.Run()
```

**requester.Work** struct contains all the option settings, including request numbers, concurrent workers, and QPS (it also contains the test result report). 

After creating an instance of **requester.Work**, then call the **Init()** method. 

```golang
func (b *Work) Init() {
	b.initOnce.Do(func() {
		b.results = make(chan *result, min(b.C*1000, maxResult))
		b.stopCh = make(chan struct{}, b.C)
	})
}
```
**Init()** method will initialize two `channel`: **results** and **stopCh**. **results** channel is used for request response communication. And **stopCh** channel is used for signal to stop the concurrent workers.

Note that there are two ways to exit from the program. The first one is the user manually stops the program, for example, by pressing **ctrl + c**. In this case, the `signal.Notify()` method from the std library can catch the signal to terminate the process. The second one is by the time **duration** option. Both of the process exiting logics are running in a `Goroutine`. 

To stop the worker, **Stop()** method will be called:

```golang
func (b *Work) Stop() {
	// Send stop signal so that workers can stop gracefully.
	for i := 0; i < b.C; i++ {
		b.stopCh <- struct{}{}
	}
}
```

What it does is sending several values to the **stopCh** channel. Note that it sends **b.C** values to the channel, which is the same as the number of concurrent workers. 

You can imagine that each worker should wait for the value from the **stopCh** channel. When the worker receives one value, it should stop sending requests. Right? Then in this way, I can stop all the concurrent workers.  Let's check our guess in the following sections.

### Concurrent Workers

In the above **main** function, you can see that **Run()** is called:

```golang
func (b *Work) Run() {
	b.Init()
	b.start = now()
	b.report = newReport(b.writer(), b.results, b.Output, b.N)
	// Run the reporter first, it polls the result channel until it is closed.
	go func() {
		runReporter(b.report)
	}()
	b.runWorkers()
	b.Finish()
}
```
There are several points worthy of discussion. In this section, let's review **runWorkers()**. And **runReporter()** and **Finish()** are related to test result reports, and we will revisit them later in this article. 

**runWorkers()** goes as follows: 

```golang
func (b *Work) runWorkers() {
	var wg sync.WaitGroup
	wg.Add(b.C)

	client := &http.Client{
		// hide details
	}

	for i := 0; i < b.C; i++ {
		go func() {
			b.runWorker(client, b.N/b.C)
			wg.Done()
		}()
	}
	wg.Wait() // block here before all workers stop
}
```

This is a very typical pattern to launch multiple `goroutine` via `sync.WaitGroup`. Each worker is created by calling **b.runWorker** in a goroutine. In this way, multiple concurrent workers can run together. 

Note that before all workers finish their tasks, **wg.Wait()** will block **Finish()** to run, which is used to report test results. And we will talk about it in the following sections.

Next step, the logic goes into **runWorker** method, and let's review how **QPS** rate limit works? 
### QPS

The core code of **runWorker** goes as follows:

```golang
func (b *Work) runWorker(client *http.Client, n int) {
	var throttle <-chan time.Time
	if b.QPS > 0 {
		throttle = time.Tick(time.Duration(1e6/(b.QPS)) * time.Microsecond)
	}
	... // hide some detail codes
	for i := 0; i < n; i++ {
		select {
		case <-b.stopCh: // receive worker stop signal from stopCh channel
			return
		default:
			if b.QPS > 0 {
				<-throttle // receive timer signal from QPS rate limite channel
			}
			b.makeRequest(client)
		}
	}
}
```

The first parameter of method **runWorker** is **client** for sending requests. We need more analysis about the second parameter **n** denoting the number of requests this worker needs to send out. When **runWorker** is called, **b.N/b.C** is passed to it. **b.N** is the total number of request need to be sent out, and **b.C** is the number of concurrent workers. **b.N** divided by **b.C** is just the number of requests for each worker. Right? 

But if the user sets the **duration** option, what is the number of requests? You can find the following logic in the **main** entry function:

```golang
	if dur > 0 {
		num = math.MaxInt32 // use MaxInt32
		if conc <= 0 {
			usageAndExit("-c cannot be smaller than 1.")
		}
	}
```
Note that when **duration** is set, the request number will be `math.MaxInt32` which is an extremely large integer. In this method, **Hey** can combine **run by duration** and **run by request number** two modes together. 

As we mentioned in the introduction part, `Hey` can support **QPS** rate limit and this strategy is written inside the **runWorker** method. Note that a `receive-only channel` **throttle** is created with `time.Tick`, which sends out a value in each time period. And the time period is defined by 

```golang
time.Duration(1e6/(b.QPS)) * time.Microsecond
```

For example, **QPS = 1000**, then the time period is 100ms, every 100ms **throttle** channel will receive a value.  

**throttle** is placed before **makeRequest()** call, and in this way we can realize the rate limit effect. 

### Stop Worker

In the **runWorker** method, you can also see the `select and case` usage. 
```golang
    select {
    case <-b.stopCh: // receive worker stop signal from stopCh channel
    	return
    // hide other code
    }
```
As we mentioned in the above section, **stopCh** channel is used to stop the worker. Right? Now you can see how it is implemented. It maps to the **Stop** method we reviewed above as follows:
```golang
	// Send stop signal so that workers can stop gracefully.
	for i := 0; i < b.C; i++ {
		b.stopCh <- struct{}{}
	}
```
`b.C` numbers of value are sent to **stopCh** channel, and there are `b.C` numbers of concurrent workers as well. Each worker can receive one value from the channel and stop running. 

### Result Report

Let's also have a quick review of how the result report work. Firstly in the **makeRequest** method, each request's result is sent to the **results** channel as follows: 

```golang
func (b *Work) makeRequest(c *http.Client) {
	// hide details
	b.results <- &result{
		// hide details
	}
}
```

And in the `runReporter` method, you can see the logic like this:

```golang
func runReporter(r *report) {
	// b.results is assign to r.results in newReport() constructor
	for res := range r.results { // receive result from results channel
		// append result to report struct
		// hide details
  	}
  	r.done <- true // send value done channel
}
```
In this case, a `for` is used to receive value from the channel. Note that **the loop will continue until the channel is closed**. So there must be one place where the channel is closed, or else the `deadlock` issue will occur. In detail, you can refer to my previous [article](https://baoqger.github.io/2020/10/26/golang-concurrent-twoways/) for more advanced explanations.  

The channel is closed in the **Finish** method like this: 

```golang
func (b *Work) Finish() {
	close(b.results)
	total := now() - b.start
	// Wait until the reporter is done.
	<-b.report.done
	b.report.finalize(total)
}
```
Please also note that how the **done** channel works. **Finish** method firstly `close` the **results** channel, then the **for** loop will break and `r.done <- true` can have chance to run. Finally **b.report.finalize()** can print the result since **<-b.report.done** is not blocked. 

























