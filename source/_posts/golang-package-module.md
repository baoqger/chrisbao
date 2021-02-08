---
title: Golang package
date: 2021-02-06 06:55:52
tags: Golang, package, workspace, vendor 
---

## Background
In this post, I'll talk about Golang package based on my learning and use experience. 

You'll learn the following topics in this post:
- Basics about Go package
- How to use and import a Go package
- Demo with real-world Go package

## Basics about Go package

#### What's Go package

Simply speaking, `Go package` provides a solution to the requirement of `code reuse`, which is an important part of software engineering. 

In Golang's official document, the definition of packages goes as following: 

>Go programs are organized into packages. A package is a collection of source files in the same directory that are compiled together. Functions, types, variables, and constants defined in one source file are visible to all other source files within the same package.

There are several critical points in the definition let's review them one by one. 

- First, one package can contain `more than one` source files. This is different from other languages, for example in `Javascript` , each source file is an independent module that exports variables to other files to import.

- Second, all the source files for a package are organized inside a directory. The package name must be the same as the directory name. 

- Third, the files inside the subdirectories should be excluded. Each subdirectory is another package. 

To have a better understanding about these three points, let's check the structure of the `net` package in the Go standard library. 

![net-package](/images/go-package-std.png)

All the `.go` source files directly under the `net` directory contain the following pacakge declaration on the top of the file: 

```golang
package net
```

This means that it is part of the net package. 

There are several subdirectories under `net` directoy, and each of these subdirectory is an independant package. For example, the `net/http` package consists of all the files inside the `http` subdirectory. If you open the files inside `http` directory, the package declaration is:  

```golang
package http
```
#### Types of Go package

Generally speaking, there are two types of packages: `library package` and `main package`. After build, the main package will be compiled into an executable file. While a library package is not self-executable, instead, it provides the utility functions.

#### Member visibility of Go package

Different from other language like Javascript, Golang package doesn't provide keyword such as `export`, `public`, `private` and so on to explicitly export members to the outside world. 

Instead, the visibility of member inside one package is determined by the casing of the first letter. If the first letter is upper case then it can be impoted by other packages. 

#### Lifecycle of package

For the library package we mentioned above, when it's imported the `init` method will be called automatically. You can do some package initialization work inside it. 

For the main pacakge, it must provide the `main` method as the entry point when it's running. 

## Use and Import Go package

Before `Go Module` was introduced, the development of Golang application is based on the `Go workspace`. In this post, I'll focus on the solutions based on `Go workspace`. Go module is another topic I will talk about in a future post. 

#### Go workspace

By convention, all your Go code and the code(or the packages) you import, must reside in a single workspace. A workspace is nothing but a directory in your file system whose path is stored in the environment variable `GOPATH`.

As a new comer into the Golang world, at the beginning the `GOPATH` workspace configuration confused me a lot. 

For example, you want to use third-party library Consol in your application. After you run 

```
go get github.com/hashicorp/consul
```

The library is installed on your local machine. The code would be cloned on disk at `$GOPATH/src/github.com/hashicorp/consul`

In your application, you will import this library in the following way:

```golang
import (
    "github.com/hashicorp/consul"
)
```

Thanks to the GOPATH mechanics, this import can be resolved on disk and Go tool can locate, build and test the code. Simply speaking, the package name maps to the real location of the package on your local machine. 

But of course, this mechanics has many limitation such as package version control, workspace constrains an so on. That's the motivation why we need Go module. 

#### Ways to import Golang package

Beside the default way, there are several ways to import a package based on your usage. 

**Import as alias**: this is useful when two packages have the same name. You can give any alias for an imported package as below:

```golang 
import (
	consulapi "github.com/hashicorp/consul/api"
)
```

**Import for side effect**: when reading source code of popular open source projects, you can see many package import in the following way:

```golang
import (
	"database/sql"

	_ "github.com/lib/pq"
)

```

It's widely used when all you need from the imported package is running the `init` method.

For example in the above case, library `pq` is imported in this way. You can check the source code for `pq` library and its `init` method call `sql.Register` method for registration as below:

```golang
func init() {
	sql.Register("postgres", &Driver{})
}
```
**Internal package**: this is an interesting feature to learn. `Internal` is a special directory name recognized by the Go tool which will prevent the package from being imported by any other packages unless both share the same ancestor directory. The packages in an `internal` directory are said to be internal packages. In detail you can refer to [this artical](https://dave.cheney.net/2019/10/06/use-internal-packages-to-reduce-your-public-api-surface).










