---
title: Hack operating system by xv6 project
date: 2020-09-22 16:58:18
tags: operating system, xv6, Unix
---

### Background

In this post, I want to introduce `xv6` which is a "simple, Unix-like operating system". 

xv6 is not only an open-source project, but also it is used for teaching purposes in MIT's Operating Systems Engineering(6.828) course as well as many other institutions. 

If you're like me, always want to learn operating system, I guess you'll face a very steep learning curve since operating system is complex. 

For learning purpose, we need an operating system project which is `not too complex` and `not  too simple`. Luckily, xv6 is just for this purpose which is simple enough to follow up as an open source project, yet still contains the important concepts and organization of Unix. 

### Resource

Since `xv6` is an open source project, you can easily find many resources online. 

Personally I recommend to use this [page](https://pdos.csail.mit.edu/6.828/2020/schedule.html), which is the latest teaching resource for the corresponding MIT course. You can find source code, examples, slides and videos there. Very helpful!

### Environment setup for xv6 project

On the MIT course's document, there are many solutions for setting up the xv6 development environment. You can follow those solutions, it will work. 

For your convenience, I make a `Dockerfile` to build the docker image which contains all the necessary dependencies to keep working on xv6. The Dockerfile goes as following:

```
FROM ubuntu:latest

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get -qq update

RUN apt-get install -y git \
                    build-essential \
                    gdb-multiarch \
                    qemu-system-misc \
                    gcc-9-riscv64-linux-gnu \
                    gcc-riscv64-linux-gnu \
                    binutils-riscv64-linux-gnu\
                    tmux \
                    qemu
```



