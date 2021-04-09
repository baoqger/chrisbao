---
title: Golang bytes.Buffer and bufio
date: 2021-04-04 17:50:14
tags: Golang, bufio, bytes, buffer
---

### Background
In this post, I will show you the usage and implementation of two Golang standard packages' : `bytes` (especially `bytes.Buffer`) and `bufio`.

These two packages are widely used in the Golang ecosystem especially works related to networking, files and other IO tasks. 

### Demo application

One good way to learn new programming knowledge is reviewing how to use it in read world applications. The following great demo application is from the open source book `Network Programming with Go by Jan Newmarch`.

For your convenience, I paste code here. This demo consists of two parts: client side and server side, two of which together form a simple directory browsing protocol. The client would be at the user end, talking to a server somewhere else. The client sends commands to the server side that allows you to list files in a directory and print the directory on the server. 

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

The implementation is easy to understand and no need to add more explanation. One interesting point is inside the `Write` function. It will first check whether the buffer has enough room for new bytes, if no then it will call   internal `grow` method to add more space. 

In fact, this is the biggest benefit you can get from `Buffer`. You don't need to manage the dynamic change of buffer length manually, `bytes.Buffer` will help you to do that. In this way you won't waste memory by setting the possible maximum length just for providing enough space. To some extend, it is similar to the **vector** in C++ language.   

### Bufio

Next, let's review how `Bufio` pacakge works. In our demo, it is used as following: 

```golang
    reader := bufio.NewReader(os.Stdin)

    for {
        line, err := reader.ReadString('\n')
        // hide other code below
    }
```

Before we dive into the details about the demo code, let's first understand what is the purpose of `bufio` package. 

First we need to understand when applications run IO operations like read or write data from or to files, network and database. It will trigger `system call` in the bottom level, which is heavy in the performance point of view.  

Buffer IO is a technique used to temporarily accumulate the results for an IO operation before transmitting it forward. This technique can increase the speed of a program by reducing the number of system calls. For example, in case you want to read data from disk byte by byte. Instead of directly reading each byte from the disk every time, with buffer IO technique, we can read a block of data into buffer once, then consumers can read data from the buffer in whatever way you want. Performance will be improved by reducing heavy system calls.

Concretely, let's review how `bufio` package do this. The Go official document goes like this:

> Package bufio implements buffered I/O. It wraps an io.Reader or io.Writer object, creating another object (Reader or Writer) that also implements the interface but provides buffering and some help for textual I/O.

Let's understand the definition by reading the source code: 

```golang
// NewReader and NewReaderSize in bufio.go
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}

func NewReaderSize(rd io.Reader, size int) *Reader {
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}
```
In our demo, we use `NewReader` which then calls `NewReaderSize` to create a new `Reader` instance. One thing need to notice is that the parameter is `io.Reader` type, which is an important interface implements only one method `Read`.

```golang
// the Reader interface in io.go file
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
In our case, we use `os.Stdin` as the function argument, which will read data from standard input. 

Then let's reivew declaration of `bufio.Reader`  which wraps `io.Reader`:
```golang
// Reader implements buffering for an io.Reader object.
type Reader struct {
	buf          []byte
	rd           io.Reader // reader provided by the client
	r, w         int       // buf read and write positions
	err          error
	lastByte     int // last byte read for UnreadByte; -1 means invalid
	lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
}
```
`bufio.Reader` has many methods defined, in our case we use `ReadString`, which will call another low-level method `ReadSlice`. 

```golang
func (b *Reader) ReadSlice(delim byte) (line []byte, err error) {
	s := 0 
	for {
		// Search buffer.
		if i := bytes.IndexByte(b.buf[b.r+s:b.w], delim); i >= 0 {
			i += s
			line = b.buf[b.r : b.r+i+1]
			b.r += i + 1
			break
		}

		if b.err != nil {
			line = b.buf[b.r:b.w]
			b.r = b.w
			err = b.readErr()
			break
		}

		if b.Buffered() >= len(b.buf) {
			b.r = b.w
			line = b.buf
			err = ErrBufferFull
			break
		}

		s = b.w - b.r 

		b.fill() // buffer is not full
	}

	if i := len(line) - 1; i >= 0 {
		b.lastByte = int(line[i])
		b.lastRuneSize = -1
	}

	return
}
```

When `buf` byte slice contains data, it will search the target value inside it. But initially `buf` is empty, it need firstly load some data, right? That is the most interesting part. The `b.fill()` is just for that. 

```golang
func (b *Reader) fill() {
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w])
		b.w -= b.r
		b.r = 0
	}

	if b.w >= len(b.buf) {
		panic("bufio: tried to fill full buffer")
	}

	// Read new data: try a limited number of times.
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:]) // call the underlying Reader
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
	b.err = io.ErrNoProgress
}
```
The data is loaded into `buf` by calling the underlying Reader,
```golang
n, err := b.rd.Read(b.buf[b.w:])
```
in our case is `os.Stdin`.

Additionally, we can define our own customized `Reader` and pass it `bufio.NewReader` to understand the buffering IO technique better as following. 

```golang
package main

import (
	"bufio"
	"fmt"
	"io"
)

// customized Reader struct
type Reader struct {
	counter int
}

func (r *Reader) Read(p []byte) (n int, err error) {
	fmt.Println("Read")
	if r.counter >= 3 { // simulate EOF
		return 0, io.EOF
	}
	s := "a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p"
	copy(p, s)
	r.counter += 1
	return len(s), nil
}

func main() {
	r := new(Reader)
	br := bufio.NewReader(r)
	for {
		token, err := br.ReadSlice(',')
		fmt.Printf("Token: %q\n", token)
		if err == io.EOF {
			fmt.Println("Read done")
			break
		}
	}
}
```

In this post, I only talked about `Reader` part of bufio, if you understand the behavior explained above clearly, it's easy to understand `Writer` quickly as well. 

