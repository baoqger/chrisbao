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

init, exit entry point. 

Typically, init_module() either registers a handler for something with the kernel, or it replaces one of the kernel functions with its own code (usually code to do something and then call the original function). The cleanup_module() function is supposed to undo whatever init_module() did, so the module can be unloaded safely.

printk source code

the path to header file

init and exit module
#### Build the module
kbuild
#### Load the module
lsmod, insmod, rmmod
#### Debug the module
dmesg
