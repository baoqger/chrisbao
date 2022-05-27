---
title: "Write a Linux firewall from scratch based on Netfilter: part two- "
date: 2022-05-05 18:06:50
tags: Linux module, Netfilter, firewall 
---

### Background

In this last [article](https://organicprogrammer.com/2022/05/04/how-to-write-a-netfilter-firewall/), we examined the basics of `Netfilter` and `Linux kernel modules` in theory. In this article, we will make our hands dirty and start implementing our mini-firewall. We will walk through the whole process step by step. This article will focus on two tasks. First, let's write our first Linux kernel module using a simple `hello world` demo. Then let's learn how to build the module(which is very different from compiling an application in the user space) and how to load it in the kernel. After understanding how to write a module, as the second task, let's write the initial version of our mini-firewall module using [Netfilter's hook architecture](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks). All right. Let's start the journey. 

### Make the first Kernel module
First, I have to admit that Linux Kernel module development is a kind of large and complex technology topic. And there are many great [online resources](https://sysprog21.github.io/lkmpg/) about it. This series of articles is focusing on developing the mini-firewall based on Netfilter, so we can't cover all the aspects of the Kernel module itself. In future articles, I'll examine more in-depth knowledge of kernel modules. 
#### Write the module

You can write the `hello world` Kernel module with a single `C` source code file as follows:  

```c
#include <linux/init.h> /* Needed for the macros */
#include <linux/kernel.h> 
#include <linux/module.h> /* Needed by all modules */

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, world\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, world\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
```
We can write a Kernel module in such an easy and simple way because the Linux Kernel does the magic for you. Remember the design philosophy of Linux(Unix): ***Design for simplicity; add complexity only where you must***. 

Let's examine several technical points worth to remark as follows: 

First, Kernel modules must have at least two functions: a "start" function which is called when the module is loaded into the kernel, and an "end" function which is called just before it is removed from the kernel. Before kernel 2.3.13, the names of these two functions are hardcoded as `init_module()` and `cleanup_module()`. But in the new versions, you can use whatever name you like for the start and end functions of a module by using the `module_init` and `module_exit` macros. The macros are defined in `include/linux/module.h` and `include/linux/init.h`. You can refer there for detailed information. 

Typically, `module_init` either registers a handler for something with the kernel (for example, the mini-firewall developed in this article), or it replaces one of the kernel functions with its own code (usually code to do something and then call the original function). The `module_exit` function is supposed to undo whatever `module_init` did, so the module can be unloaded safely.

Second, `printk` function provides similar behaviors to `printf`, which accepts the `format string` as the first argument. The `printk` function prototype goes as follows:

```c
int printk(const char *fmt, ...);
```

`printk` function allows a caller to specify `log level` to indicate the type and importance of the message being sent to the kernel message log. For example, in the above code, the log level `KERN_INFO` is specified by prepending to the format string. In C programming, this syntax is called [`string literal concatenation`](https://en.wikipedia.org/wiki/String_literal#String_literal_concatenation). (In other high-level programming languages, string concatenation is generally done with `+` operator). For the function `printk` and `log level`, you can find more information in `include/linux/kern_levels.h` and `include/linux/printk.h`.   

Note: The path to header files for Linux kernel module development is different from the one you often used for the application development. Don't try to find the header file inside */usr/include/linux*, instead please use the following path */lib/modules/\`uname -r\`/build/include/linux* (`uname -r` command returns your kernel version).
#### Build the module
kbuild
#### Load the module
lsmod, insmod, rmmod
#### Debug the module
dmesg
