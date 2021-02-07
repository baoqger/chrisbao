---
title: Golang package
date: 2021-02-06 06:55:52
tags: Golang, package, workspace, vendor 
---

### Background
In this post, I'll talk about Golang package based on my learning and use experience. 

You'll learn the following topics in this post:
- Basics about Go package
- How to use and import a Go package
- Demo with real-world Go package

### Basics about Go package
package definition, package types, package lifecycle

Simply speaking, `Go package` provides a solution to the requirement of `code reuse`, which is an important part of software engineering. 

In Golang's official document, the definition of package goes as following: 

>Go programs are organized into packages. A package is a collection of source files in the same directory that are compiled together. Functions, types, variables, and constants defined in one source file are visible to all other source files within the same package.

There are several critical points in the definition,let's review them one by one. 

First, one package can contain `more than one` source files. This is different from other languages, for example in `Javascript` , each source file is an independent module which can export variables to other files to import.

Second, all the source files for a package are organized inside a directory. Package name must be the same as the directory name. 

Third, the files inside the subdirectories should be excluded. Each subdirectory is another package. 

### Import Go package
4 ways to import Go package

### Vendor Directory




