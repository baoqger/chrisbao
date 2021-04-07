---
title: golang-bytes-buffer-bufio
date: 2021-04-04 17:50:14
tags: Golang, bufio, bytes, buffer
---

### Background
In this post, I will show you two Golang standard libraries' usage and implemention. They are: `bytes` (especially `bytes.Buffer`) and `bufio`.

These two libraries are widely used in the Golang ecosystem when you're working on networking, files and other `IO` tasks. 

### Demo application

One good way to learn new programming knowledge is reviewing how to use it in read world applications. The following great demo application is from the open source book `Network Programming with Go by Jan Newmarch`.

For your convenience, I paste code here. This demo consists of two parts: client side and server side, two of which together form a simple directory browsing protocol. The client would be at the user end, talking to a server somewhere else. Client sends commands to server side that allows you to list files in a directory and change and print the directory on the server. 

First is the client side program: 

```golang
package main

import (
    "fmt"
    "net"
    "os"
    "bufio"
    "strings"
    "bytes"
)

// strings used by the user interface
const (
    uiDir  = "dir"
    uiCd   = "cd"
    uiPwd  = "pwd"
    uiQuit = "quit"
)

// strings used across the network
const (
    DIR = "DIR"
    CD  = "CD"
    PWD = "PWD"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Usage: ", os.Args[0], "host")
        os.Exit(1)
    }

    host := os.Args[1]

    conn, err := net.Dial("tcp", host+":1202")
    checkError(err)

    reader := bufio.NewReader(os.Stdin)
    for {
        line, err := reader.ReadString('\n')
        // lose trailing whitespace
        line = strings.TrimRight(line, " \t\r\n")
        if err != nil {
            break
        }

        // split into command + arg
        strs := strings.SplitN(line, " ", 2)
        // decode user request
        switch strs[0] {
        case uiDir:
            dirRequest(conn)
        case uiCd:
            if len(strs) != 2 {
                fmt.Println("cd <dir>")
                continue
            }
            fmt.Println("CD \"", strs[1], "\"")
            cdRequest(conn, strs[1])
        case uiPwd:
            pwdRequest(conn)
        case uiQuit:
            conn.Close()
            os.Exit(0)
        default:
            fmt.Println("Unknown command")
        }
    }
}

func dirRequest(conn net.Conn) {
    conn.Write([]byte(DIR + " "))

    var buf [512]byte
    result := bytes.NewBuffer(nil)
    for {
        // read till we hit a blank line
        n, _ := conn.Read(buf[0:])
        result.Write(buf[0:n])
        length := result.Len()
        contents := result.Bytes()
        if string(contents[length-4:]) == "\r\n\r\n" {
            fmt.Println(string(contents[0 : length-4]))
            return
        }
    }
}

func cdRequest(conn net.Conn, dir string) {
    conn.Write([]byte(CD + " " + dir))
    var response [512]byte
    n, _ := conn.Read(response[0:])
    s := string(response[0:n])
    if s != "OK" {
        fmt.Println("Failed to change dir")
    }
}

func pwdRequest(conn net.Conn) {
    conn.Write([]byte(PWD))
    var response [512]byte
    n, _ := conn.Read(response[0:])
    s := string(response[0:n])
    fmt.Println("Current dir \"" + s + "\"")
}

func checkError(err error) {
    if err != nil {
        fmt.Println("Fatal error ", err.Error())
        os.Exit(1)
    }
}
```
##### **`client.go`**

Then is server side code: 

```golang
package main

import (
    "fmt"
    "net"
    "os"
)

const (
    DIR = "DIR"
    CD  = "CD"
    PWD = "PWD"
)

func main() {

    service := "0.0.0.0:1202"
    tcpAddr, err := net.ResolveTCPAddr("tcp", service)
    checkError(err)

    listener, err := net.ListenTCP("tcp", tcpAddr)
    checkError(err)

    for {
        conn, err := listener.Accept()
        if err != nil {
            continue
        }
        go handleClient(conn)
    }
}

func handleClient(conn net.Conn) {
    defer conn.Close()

    var buf [512]byte
    for {
        n, err := conn.Read(buf[0:])
        if err != nil {
            conn.Close()
            return
        }

        s := string(buf[0:n])
        // decode request
        if s[0:2] == CD {
            chdir(conn, s[3:])
        } else if s[0:3] == DIR {
            dirList(conn)
        } else if s[0:3] == PWD {
            pwd(conn)
        }

    }
}

func chdir(conn net.Conn, s string) {
    if os.Chdir(s) == nil {
        conn.Write([]byte("OK"))
    } else {
        conn.Write([]byte("ERROR"))
    }
}

func pwd(conn net.Conn) {
    s, err := os.Getwd()
    if err != nil {
        conn.Write([]byte(""))
        return
    }
    conn.Write([]byte(s))
}

func dirList(conn net.Conn) {
    defer conn.Write([]byte("\r\n"))

    dir, err := os.Open(".")
    if err != nil {
        return
    }

    names, err := dir.Readdirnames(-1)
    if err != nil {
        return
    }
    for _, nm := range names {
        conn.Write([]byte(nm + "\r\n"))
    }
}

func checkError(err error) {
    if err != nil {
        fmt.Println("Fatal error ", err.Error())
        os.Exit(1)
    }
}
```
##### **`server.go`**

### Bytes.Buffer

Based on the above demo, let's review how `Bytes.Buffer` is used. 

According to Go official document:  

> Package bytes implements functions for the manipulation of byte slices.
> A Buffer is a variable-sized buffer of bytes with Read and Write methods.

The `bytes` package itself is easy to understand, which provides functionalities to manipulate byte slice. The concern is `bytes.Buffer`, what benefits can we get by using it? Let's review the demo code where it is used.

```golang
func dirRequest(conn net.Conn) {
    conn.Write([]byte(DIR + " "))

    var buf [512]byte
    result := bytes.NewBuffer(nil)
    for {
        // read till we hit a blank line
        n, _ := conn.Read(buf[0:])
        result.Write(buf[0:n])
        length := result.Len()
        contents := result.Bytes()
        if string(contents[length-4:]) == "\r\n\r\n" {
            fmt.Println(string(contents[0 : length-4]))
            return
        }
    }
}
```
The above code block is from `client.go` part. And the scenario is: the client send `DIR` command to server side, server run this `DIR` command which will return contents of current directory. Client and server use `conn.Read` and `conn.Write` to communicate with each other. The client keeps reading data in a `for` loop until all the data is consumed which is marked by two continuous `\r\n` strings. 

In this case, a new `bytes.Buffer` object is created by calling `NewBuffer` method and three other member methods are called: `Write`, `Len` and `Bytes`. Let's review their source code: 

```golang
type Buffer struct {
	buf      []byte 
	off      int    
	lastRead readOp 
}

func (b *Buffer) Write(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	m, ok := b.tryGrowByReslice(len(p))
	if !ok {
		m = b.grow(len(p))
	}
	return copy(b.buf[m:], p), nil
}

func (b *Buffer) Len() int { 
    return len(b.buf) - b.off 
}

func (b *Buffer) Bytes() []byte { 
    return b.buf[b.off:] 
}
```

The implementation is easy to understand and no need to add more explanation. One interesting point is inside the `Write` function. It will first check whether the buffer has enough room to new bytes, if no then it will call   internal `grow` method to add more space. 

This is in fact the biggest benefit you can get from `Buffer`. You don't need to manage the dynamic change of buffer length manually, `bytes.Buffer` will help you to do that. In this way you won't waste memory by setting the possible maximum length just for providing enough space. To some extend, it is similar to the **vector** in C++ language.   




Demo application: directory-browsering-protocol

bytes.buffer variable length byte slice.

bufio => reduce io system call
customer Reader to show the buffering io behavior
ReadSlice implementation call underlying reader.read
