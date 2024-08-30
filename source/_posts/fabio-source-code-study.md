---
title: Fabio source code study part 1
date: 2021-01-29 22:09:56
tags: Fabio, Golang, Source code
keywords: Fabio, Golang, Source code
---
### Background

In this two-part blog series, I want to share the lessons learned from reading the soruce code of the project `Fabio`. In my previous blog, I shared with you how to use Fabio as a load balancing in the micro services applicatoins, in detail you can refer to this [article](https://baoqger.github.io/2020/12/30/golang-load-balancing-fabio/).  


Since `Fabio` is not a tiny project, it's hard to cover everything inside this project. I will mainly focus on two aspects: firstly in the **architecture design  level**, I will study how it can work as a load balancer without any configuration file (Part one), and secondly in the **language level**, I want to summarize the best practice of writing Golang programs by investigating which features of Golang it uses and how it uses (Part two). 

### Fabio architecture design

Let's start by introducing some background about Fabio. Following paragraph is from its official document: 

> Fabio is an HTTP and TCP reverse proxy that configures itself with data from Consul. Traditional load balancers and reverse proxies need to be configured with a config file. 

If you're familiar with other load balancer service such as `Nginx`, it will be easy for you to understand how Fabio is different and why it seems interestring. 

For example, if you're using `Nginx` as your load balancer, you need to maintain a config file where the routing rules need to be defined as below 

```nginx
server {
    location / {
        root /data/www;
    }

    location /images/ {
        root /data;
    }
}
```

But Fabio is a zero-conf load balancer. Cool, right? Let's review the design and code to uncover the secrets under the hood. 

![load-balancing](/images/fabio.png)

Simply speaking, Fabio's design can be divided into two parts: **Consul monitor** and **proxy**. Consul monitor forms and updates a `route table` by watching the data stored in Consul, and proxy distributes the request to target service instance based on the `route table`.

#### main function

```golang
func main() {
	logOutput := logger.NewLevelWriter(os.Stderr, "INFO", "2017/01/01 00:00:00 ")
	log.SetOutput(logOutput)

	cfg, err := config.Load(os.Args, os.Environ())
	if err != nil {
		exit.Fatalf("[FATAL] %s. %s", version, err)
	}
	if cfg == nil {
		fmt.Println(version)
		return
	}

	log.Printf("[INFO] Setting log level to %s", logOutput.Level())
	if !logOutput.SetLevel(cfg.Log.Level) {
		log.Printf("[INFO] Cannot set log level to %s", cfg.Log.Level)
	}

	log.Printf("[INFO] Runtime config\n" + toJSON(cfg))
	log.Printf("[INFO] Version %s starting", version)
	log.Printf("[INFO] Go runtime is %s", runtime.Version())

	WarnIfRunAsRoot(cfg.Insecure)

	var prof interface {
		Stop()
	}
	if cfg.ProfileMode != "" {
		var mode func(*profile.Profile)
		switch cfg.ProfileMode {
		case "":
			// do nothing
		case "cpu":
			mode = profile.CPUProfile
		case "mem":
			mode = profile.MemProfile
		case "mutex":
			mode = profile.MutexProfile
		case "block":
			mode = profile.BlockProfile
		case "trace":
			mode = profile.TraceProfile
		default:
			log.Fatalf("[FATAL] Invalid profile mode %q", cfg.ProfileMode)
		}

		prof = profile.Start(mode, profile.ProfilePath(cfg.ProfilePath), profile.NoShutdownHook)
		log.Printf("[INFO] Profile mode %q", cfg.ProfileMode)
		log.Printf("[INFO] Profile path %q", cfg.ProfilePath)
	}

	exit.Listen(func(s os.Signal) {
		atomic.StoreInt32(&shuttingDown, 1)
		proxy.Shutdown(cfg.Proxy.ShutdownWait)
		if prof != nil {
			prof.Stop()
		}
		if registry.Default == nil {
			return
		}
		registry.Default.DeregisterAll()
	})

	initMetrics(cfg)
	initRuntime(cfg)
	initBackend(cfg)

	trace.InitializeTracer(&cfg.Tracing)

	startAdmin(cfg)

	go watchNoRouteHTML(cfg)

	first := make(chan bool)
	go watchBackend(cfg, first)
	log.Print("[INFO] Waiting for first routing table")
	<-first

	startServers(cfg)

	WarnIfRunAsRoot(cfg.Insecure)

	exit.Wait()
	log.Print("[INFO] Down")
}
```

The `main` function defines Fabio's workflow. To understand how Fabio works, we only need to focus on three points: 
- **initBackend()** and **watchBackend()**: these two functions contain Consul monitoring logic. 
- **startServers()**: this function is responsible to create the network proxy.

#### Consul monitoring

First, let's review the `initBackend` function: 

```golang
func initBackend(cfg *config.Config) {
	var deadline = time.Now().Add(cfg.Registry.Timeout)
	var err error
	for {
		switch cfg.Registry.Backend {
		case "file":
			registry.Default, err = file.NewBackend(&cfg.Registry.File)
		case "static":
			registry.Default, err = static.NewBackend(&cfg.Registry.Static)
		case "consul":
			registry.Default, err = consul.NewBackend(&cfg.Registry.Consul)
		case "custom":
			registry.Default, err = custom.NewBackend(&cfg.Registry.Custom)
		default:
			exit.Fatal("[FATAL] Unknown registry backend ", cfg.Registry.Backend)
		}

		if err == nil {
			if err = registry.Default.Register(nil); err == nil {
				return
			}
		}
		log.Print("[WARN] Error initializing backend. ", err)

		if time.Now().After(deadline) {
			exit.Fatal("[FATAL] Timeout registering backend.")
		}

		time.Sleep(cfg.Registry.Retry)
		if atomic.LoadInt32(&shuttingDown) > 0 {
			exit.Exit(1)
		}
	}
}
```
This function is not hard to understand. Fabio supports various modes: `file`, `static`, `consul` and `custom`, and will select one mode to use based on the detailed condition inside the `cfg` parameter. In our case, we only need to focus on the `consul` mode. 

Next let's review `watchBackend()` function to check how it keeps watching consul's data.

```golang
func watchBackend(cfg *config.Config, first chan bool) {
	var (
		nextTable   string
		lastTable   string
		svccfg      string
		mancfg      string
		customBE    string
		once        sync.Once
		tableBuffer = new(bytes.Buffer) // fix crash on reset before used (#650)
	)

	switch cfg.Registry.Backend {
	case "custom":
		svc := registry.Default.WatchServices()
		for {
			customBE = <-svc
			if customBE != "OK" {
				log.Printf("[ERROR] error during update from custom back end - %s", customBE)
			}
			once.Do(func() { close(first) })
		}
	// all other backend types
	default:
		svc := registry.Default.WatchServices()
		man := registry.Default.WatchManual()

		for {
			select {
			case svccfg = <-svc:
			case mancfg = <-man:
			}
			// manual config overrides service config - order matters
			tableBuffer.Reset()
			tableBuffer.WriteString(svccfg)
			tableBuffer.WriteString("\n")
			tableBuffer.WriteString(mancfg)

			if nextTable = tableBuffer.String(); nextTable == lastTable {
				continue
			}
			aliases, err := route.ParseAliases(nextTable)
			if err != nil {
				log.Printf("[WARN]: %s", err)
			}
			registry.Default.Register(aliases)
			t, err := route.NewTable(tableBuffer)
			if err != nil {
				log.Printf("[WARN] %s", err)
				continue
			}
			route.SetTable(t)
			logRoutes(t, lastTable, nextTable, cfg.Log.RoutesFormat)
			lastTable = nextTable
			once.Do(func() { close(first) })
		}
	}
}
```
Firstly in line 24, we need to understand `registry.Default.WatchServices()`. Since `initBackend` function already decided we're using Consul mode, so we need to check the `WatchServices()` function inside the `Consul` package as following:

```golang
package consul

func (b *be) WatchServices() chan string {
	log.Printf("[INFO] consul: Using dynamic routes")
	log.Printf("[INFO] consul: Using tag prefix %q", b.cfg.TagPrefix)

	m := NewServiceMonitor(b.c, b.cfg, b.dc)
	svc := make(chan string)
	go m.Watch(svc)
	return svc
}
```
The return value is `svc` which is just a string typed `channel`. And `svc` channel is passed into goroutine `go m.watch()` as an argument. This is a very typical usage in Golang programming where two goroutines need to communicate with each other via the channel. Let's go on and check the `Watch` function:

```golang
func (w *ServiceMonitor) Watch(updates chan string) {
	var lastIndex uint64
	var q *api.QueryOptions
	for {
		if w.config.PollInterval != 0 {
			q = &api.QueryOptions{RequireConsistent: true}
			time.Sleep(w.config.PollInterval)
		} else {
			q = &api.QueryOptions{RequireConsistent: true, WaitIndex: lastIndex}
		}
		checks, meta, err := w.client.Health().State("any", q)
		if err != nil {
			log.Printf("[WARN] consul: Error fetching health state. %v", err)
			time.Sleep(time.Second)
			continue
		}
		log.Printf("[DEBUG] consul: Health changed to #%d", meta.LastIndex)
		// determine which services have passing health checks
		passing := passingServices(checks, w.config.ServiceStatus, w.strict)
		// build the config for the passing services
		updates <- w.makeConfig(passing)
		// remember the last state and wait for the next change
		lastIndex = meta.LastIndex
	}
}
```

You can see `updates <- w.makeConfig(passing)` in Line 21, it just sends a message into the channel.  

Another interestring point is `w.client.Health().State("any", q)` in line 11. This is one API provided in the `consul/api` package. If you check the implementation of it, you'll find out in fact it just sends a HTTP get request to this endpoint `/v1/health/state/` of Consul service which will return all the health check status for the services registered in Consul. 

And the above logic runs inside a `for` loop, in this way Fabio keeps sending request to query the latest status from Consul. If new services are discovered, then the status will be updated dynamically as well, no need to restart Fabio. 

So far you should understand how Fabio can work as a load balancer with any hardcoded routing config.  

Let's go back to the `watchBackend` function to continue the analysis. 

After debugging, I find the message passed via the `svc` channel follows the following format:

```
[route add demo-service /helloworld http://127.0.0.1:5000/ route add demo-service /foo http://127.0.0.1:5000/]
```

Next step is converting this string message into the route table. 

In line 46 and 51 of `watchBackend` function, you can find these two lines of code:

``` golang
t, err := route.NewTable(tableBuffer) // line 46

...

route.SetTable(t) // line 51
```

Everything will be clear after you check the implementation of the `route` package. 

`route.NewTable()` function returns a `Table` type value which is map in fact. And the `Table` type declaration goes as following: 

```golang
// Table contains a set of routes grouped by host.
// The host routes are sorted from most to least specific
// by sorting the routes in reverse order by path.
type Table map[string]Routes

// Routes stores a list of routes usually for a single host.
type Routes []*Route

// Route maps a path prefix to one or more target URLs.
// routes can have a weight value which describes the
// amount of traffic this route should get. You can specify
// that a route should get a fixed percentage of the traffic
// independent of how many instances are running.
type Route struct {
	// Host contains the host of the route.
	// not used for routing but for config generation
	// Table has a map with the host as key
	// for faster lookup and smaller search space.
	Host string

	// Path is the path prefix from a request uri
	Path string

	// Targets contains the list of URLs
	Targets []*Target

	// wTargets contains targets distributed according to their weight
	wTargets []*Target

	// total contains the total number of requests for this route.
	// Used by the RRPicker
	total uint64

	// Glob represents compiled pattern.
	Glob glob.Glob
}
```

That's all for the consul monitor part. Simply speaking, Fabio keeps looping the latest service status from Consul and process the status information into a routing table. 

#### Proxy

The second part is about network proxy, which is easier to understand than the first part. 

Fabio supports various network protocols, but in this post let's focus on `HTTP/HTTPS` case. In side the `main.go` file, you can find the following function:

```golang
func newHTTPProxy(cfg *config.Config) http.Handler {
	var w io.Writer

	//Init Glob Cache
	globCache := route.NewGlobCache(cfg.GlobCacheSize)

	switch cfg.Log.AccessTarget {
	case "":
		log.Printf("[INFO] Access logging disabled")
	case "stdout":
		log.Printf("[INFO] Writing access log to stdout")
		w = os.Stdout
	default:
		exit.Fatal("[FATAL] Invalid access log target ", cfg.Log.AccessTarget)
	}

	format := cfg.Log.AccessFormat
	switch format {
	case "common":
		format = logger.CommonFormat
	case "combined":
		format = logger.CombinedFormat
	}

	l, err := logger.New(w, format)
	if err != nil {
		exit.Fatal("[FATAL] Invalid log format: ", err)
	}

	pick := route.Picker[cfg.Proxy.Strategy]
	match := route.Matcher[cfg.Proxy.Matcher]
	notFound := metrics.DefaultRegistry.GetCounter("notfound")
	log.Printf("[INFO] Using routing strategy %q", cfg.Proxy.Strategy)
	log.Printf("[INFO] Using route matching %q", cfg.Proxy.Matcher)

	newTransport := func(tlscfg *tls.Config) *http.Transport {
		return &http.Transport{
			ResponseHeaderTimeout: cfg.Proxy.ResponseHeaderTimeout,
			MaxIdleConnsPerHost:   cfg.Proxy.MaxConn,
			Dial: (&net.Dialer{
				Timeout:   cfg.Proxy.DialTimeout,
				KeepAlive: cfg.Proxy.KeepAliveTimeout,
			}).Dial,
			TLSClientConfig: tlscfg,
		}
	}

	authSchemes, err := auth.LoadAuthSchemes(cfg.Proxy.AuthSchemes)

	if err != nil {
		exit.Fatal("[FATAL] ", err)
	}

	return &proxy.HTTPProxy{
		Config:            cfg.Proxy,
		Transport:         newTransport(nil),
		InsecureTransport: newTransport(&tls.Config{InsecureSkipVerify: true}),
		Lookup: func(r *http.Request) *route.Target {
			t := route.GetTable().Lookup(r, r.Header.Get("trace"), pick, match, globCache, cfg.GlobMatchingDisabled)
			if t == nil {
				notFound.Inc(1)
				log.Print("[WARN] No route for ", r.Host, r.URL)
			}
			return t
		},
		Requests:    metrics.DefaultRegistry.GetTimer("requests"),
		Noroute:     metrics.DefaultRegistry.GetCounter("notfound"),
		Logger:      l,
		TracerCfg:   cfg.Tracing,
		AuthSchemes: authSchemes,
	}
}
```

The return value's type is `http.Handler`, which is an **interface** defined inside Go standard library as following:

```golang
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

And the actual return value's type is `proxy.HTTPProxy` which is a `struct` implementing the `ServeHTTP` method. You can find the code inside the `proxy` package in Fabio repo. 

```golang
type HTTPProxy struct {
	...
}

func (p *HTTPProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	...
}
```

Another point needs to be mentioned is `Lookup` field of `HTTPProxy` struct:

```golang
Lookup: func(r *http.Request) *route.Target {
	t := route.GetTable().Lookup(r, r.Header.Get("trace"), pick, match, globCache, cfg.GlobMatchingDisabled)
	if t == nil {
		notFound.Inc(1)
		log.Print("[WARN] No route for ", r.Host, r.URL)
	}
	return t
}
```

You don't need to understand the details, just pay attention to `route.GetTable()` which is the routing table mentioned above. Consul monitor maintains the table and proxy consumes the table. That's it.

In this article which is part one of this blog series , you learned how Fabio can serve as a load balancer without any config files by reviewing the design and reading the source code. 

In part two, let's review how Golang was used and try to summarize the best practise of wrting Golang programs.

