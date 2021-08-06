---
title: New way to set up Linux development environment in Windows with WSL
date: 2021-08-06 09:19:41
tags: Linux, system programming, Windows Subsystem for Linux
---

### Background

I like system programming which can allow you to touch more software development skills in the bottom level.  

Linux is the perfect platform when you want to do system programming. But if you're using a computer running Windows on it, then you have to spend some time to set up the Linux development environment. Generally speaking there are two traditional ways to do that: `virtual machine` and `dualboot`, both need some effort. Or you can try to do that with `container` technology, for example, I once shared one [article](https://baoqger.github.io/2020/07/28/linux-cpp-docker/) about how to do it with `Docker`. 

In this article, I will introduce a new and easier way to do this without too much overhead. 

### Windows Subsystem for Linux

The new way is `Windows Subsystem for Linux (WSL)`. I have to admit that the operating system is complex and difficult so for now I don't know how Microsoft make `WSL` works. In details, you can refer to this [article](https://devblogs.microsoft.com/commandline/a-deep-dive-into-how-wsl-allows-windows-to-access-linux-files/) to learn how `WSL` allows Windows to access Linux files. In this article let's focus on how to set it up and what kind of benefits it can provide to developers.

Based on the [official document](https://docs.microsoft.com/en-us/windows/wsl/about), with `WSL` you can

- Run common command-line tools such as `grep`, `sed`, `awk`.
- Run Bash shell scripts and GNU/Linux command-line applications including:
  - Tools: vim, emacs, tmux.
  - Languages: NodeJS, Javascript, Python, Ruby, C/C++, C# & F#, Rust, Go, etc.
  - Services: SSHD, MySQL, Apache, lighttpd, MongoDB, PostgreSQL.
- Install additional software using your own GNU/Linux distribution package manager.

With these conditions, you can set up a completed Linux development environment. 

### Install WSL

For detail steps to install `WSL`, you can find it on the official document. Based on my experience, I follow the document to download and install Linux Ubuntu distribution smoothly, which is much easier than settig the virtual machine.

### File mount

By default, you can also access your local machine’s file system from within the Linux Bash shell. Since your local drives are mounted under the **/mnt** folder of the subsystem. 

In this way, you can develop the code with the productivity tools in Windows and build it in Linux environment.


### Network

This is another convenient point. `WSL` shares the IP address of Windows, as it is running on Windows.
As such you can access any ports on localhost e.g. if you had a web server running on port 8080, you could access it just by visiting **http://localhost:8080** into your Windows browser.

### Set up the development environment

After install the Ubuntu system, I also install tools to prepare the development environment. For example, `GCC` to develop C language program as below. 

<img src="/images/linux-gcc.png" title="gcc in Linux" width="800px" height="600px">

### Summary
Based on my testing and experience, `WSL` can save developers' time to set up Linux environment.  
