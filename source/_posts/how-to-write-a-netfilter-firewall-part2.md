---
title: "Write a Linux firewall from scratch based on Netfilter: part two- hello world module"
date: 2022-05-05 18:06:50
tags: Linux module, Netfilter, firewall 
---

### Background

In this last [article](https://organicprogrammer.com/2022/05/04/how-to-write-a-netfilter-firewall/), we examined the basics of `Netfilter` and `Linux kernel modules` in theory. Starting from this article, we will make our hands dirty and start implementing our mini-firewall. We will walk through the whole process step by step. In this article, let's write our first Linux kernel module using a simple `hello world` demo. Then let's learn how to build the module(which is very different from compiling an application in the user space) and how to load it in the kernel. After understanding how to write a module, in the next article, let's write the initial version of our mini-firewall module using [Netfilter's hook architecture](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks). All right. Let's start the journey. 

### Make the first Kernel module
First, I have to admit that Linux Kernel module development is a kind of large and complex technology topic. And there are many great [online resources](https://sysprog21.github.io/lkmpg/) about it. This series of articles is focusing on developing the mini-firewall based on Netfilter, so we can't cover all the aspects of the Kernel module itself. In future articles, I'll examine more in-depth knowledge of kernel modules. 
#### Write the module

You can write the `hello world` Kernel module with a single `C` source code file `hello.c` as follows:  

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

Next, let's build this hello-world kernel module.
#### Build the module
The way to build a kernel module is a little different from how to build a user-space application. The efficient solution to build kernel image and its modules is `Kernel Build System(Kbuild)`. 

`Kbuild` is a complex topic and I won't explain it in too much detail here. Simply speaking, `Kbuild` allows you to create highly customized kernel binary images and modules. Technically, each subdirectory contains a `Makefile` compiling only the source code files in its directory. And a top-level Makefile recursively executes each subdirectory's Makefile to generate the binary objects. And you can control which subdirectories are included by defining `config files`. In detail, you can refer to other [documents](https://www.linuxjournal.com/content/kbuild-linux-kernel-build-system). 

The following is the Makefile for the `hello world` module: 
```Makefile
obj-m += hello.o
PWD := $(CURDIR)
all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

The `make -C dir` command changes to directory dir before reading the makefiles or doing anything else. The top-level Makefile in */lib/modules/$(shell uname -r)/build* will be used. You can find that command `make M=dir modules` is used to make all modules in specified dir.

And in the module-level Makefile, the `obj-m` syntax tells `kbuild` system to build `module_name.o` from `module_name.c`, and after linking, will result in the kernel module `module_name.ko`. In our case, the module name is `hello`.

The build process goes as follows: 

```bash
chrisbao:~/develop/kernel/hello-1$ sudo make
make -C /lib/modules/4.15.0-176-generic/build M=/home/DIR/jbao6/develop/kernel/hello-1  modules
make[1]: Entering directory '/usr/src/linux-headers-4.15.0-176-generic'
  CC [M]  /home/DIR/jbao6/develop/kernel/hello-1/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/DIR/jbao6/develop/kernel/hello-1/hello.mod.o
  LD [M]  /home/DIR/jbao6/develop/kernel/hello-1/hello.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.15.0-176-generic'
```
After the build, you can get several new files in the same directory: 
```bash
chrisbao:~/develop/kernel/hello-1$ ls
hello.c  hello.ko  hello.mod.c  hello.mod.o  hello.o  Makefile  modules.order  Module.symvers
```
The file ends with `.ko` is the kernel module. You can ignore other files now, I will write another article later to have a deep discussion about the kernel module system. 

#### Load the module
With the `file` command, you can note that the kernel module is an `ELF(Executable and Linkable Format)` format file. ELF files are typically the output of a compiler or linker and are a binary format. 

```bash
chrisba:~/develop/kernel/hello-1$ file hello.ko
hello.ko: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), BuildID[sha1]=f0da99c757751e7e9f9c4e55f527fb034a0a4253, not stripped
```

Next step, let's try to install and remove the module dynamically. You need to know the following three commands: 

- *lsmod*: shows the list of kernel modules currently loaded.
- *insmod*: inserts a module into the Linux Kernel by running `sudo insmod module_name.ko`
- *rmmod*: removes a module from the Linux Kernel by running `sudo rmmod module_name`

Since the `hello world` module is quite simple, you can easily install and remove the module as you wish. I will not show the detailed commands here and leave it to the readers. 

**Note**: It doesn't mean that you can easily install and remove any kernel module without any issues. If the module you are loading has bugs, the entire system can crash. 
#### Debug the module

Next step, let's prove that the `hello world` module is installed and removed as expected. We will use `dmesg` command. `dmesg` (diagnostic messages) can print the messages in the `kernel ring buffer`. 

First, a [`ring buffer`](https://en.wikipedia.org/wiki/Circular_buffer) is a data structure that uses a single, fixed-size buffer as if it were connected end-to-end. The `kernel ring buffer` is a ring buffer that records messages related to the operation of the kernel. As we mentioned above, the kernel logs printed by the `printk` function will be sent to the kernel ring buffer. 

We can find the messages produced by our module with command `dmesg | grep world` as follows:

```bash
chrisbao:~$ dmesg | grep world

[2147137.177254] Hello, world
[3281962.445169] Goodbye, world
[3282008.037591] Hello, world
[3282054.921824] Goodbye, world
```

Now you can see that the `hello world` is loaded into the kernel correctly. And it can be removed dynamically as well. Great. 

### Summary

In this article, we examine how to write a kernel module, how to build it and how to install it into the kernel dynamically. Next article we can work on the mini-firewall as a `Netfilter` module. 


