---
title: How to write a Golang HTTP server with Linux system calls 
date: 2021-07-31 10:15:25
tags: HTTP, system call, Linux, Golang
---

### Background

`HTTP` is everywhere. As a software engineer, you're using the `HTTP` protocol every day. Starting an `HTTP` server will be an easy task if you're using any modern language or framework. For example, in Golang you can do that with the following lines of code:

```golang
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/hi", func(w http.ResponseWriter, r *http.Request){
        fmt.Fprintf(w, "Hi")
    })

    log.Fatal(http.ListenAndServe(":8081", nil))
}
```

You can finish the job easily because `net/http` package implements the `HTTP` protocol completely. Without `net/http` package how can you do that? That's the target of this article. 

**Note**: 


