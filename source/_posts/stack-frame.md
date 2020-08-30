---
title: Understand stack memory management
date: 2020-08-19 04:54:00
tags: memory management, stack
---

### Table of Contents 
First I have to admit that memory management is a big( and important) topic in operating system. This post can't cover everything related to it. 

In this post, I want to share something interesting I learned about stack memory management. Especially understand how the stack frame of function call works, which's the most important part of stack memory. I will explain the mechanism in detail with examples and diagrams.

Briefly speaking, the contents of this post is:

- Memory layout of a process
- Stack memory contents
- CPU register related to stack memory management
- Stack frame of function call and return mechanism

To understand the concepts and machanisms deeply, a few assembly code will be shown in this post, you don't have to know assembly lauguage as an expert for reading this post, I will add comments for these assembly codes to make explain what's their functionalities.


### Stack Memory Management Basic

#### Memory Layout of Process

Before we talk about Stack memory management, it's necessary to understand `memory layout` of a process.

When you create a program, it is just some Bytes of data which is stored in the disk. When a program executes then it starts loading into memory and become a live entity. In a more specify way, some memory (virtual memory) is allocated to each process in execution for its usage by operating system. The memory assigned to a process is called process's `virtual address space`( short for VAS). And memory layout use diagrams to help you understand the concept of process virtual address space easily.

There are some awesome posts on the internet about [memory layout](https://www.includehelp.com/operating-systems/memory-layout-of-a-process.aspx). And the most two critical sections are: Stack and Heap. Simply speaking, `Stack` is the memory area which is used by each process to store the local variables, passed arguments and other information when a function is called. This is the topic of this post, you'll know more detailed in the following sections. While `Heap` segment is the one which is used for dynamic memory allocation, this is another more complex topic out of this post's scope. 

#### Stack Memory Basics
Stack Memory is just memory region in each process's virtual address space where `stack` data structure (Last in, first out) is used to store the data. As we mentioned above, when a new function call is invoked, then a frame of data is pushed to stack memory and when the function call returns then the frame is removed from the stack memory. This is **Stack Frame** which represent a function call and its argument data. And every function has its own stack frame.

Let's say in your program there are multiple functions call each other in order. For example, `main() -> f1() -> f2() -> f3()`, `main` function calls function one `f1()`, then function one calls function two `f2()`, finally function two calls function three `f3()`. So based on the Last in first out rule, the stack memory will be like as below: 

![stack-frame](/images/stack_frames.png)

**Note**: the top of the stack is the bottom part of the image. Don't feel confused for that. 

#### Stack memory contents

After understand stack frames, now let's dive deep into each stack frame to discover its contents.

Each stack frame contains 4 parts:

- Parameter passed to the callee function
- Return address of the caller function
- Base pointer
- Local variables of the callee function


![stack-contents](/images/stack_contents.png)

Caller and callee is very easy to understand. Let's say `main() -> f1()`, then caller function is `main()` while callee function is `f1()`. Right? 

For the above 4 four parts, there are something need to emphasis. Firstly the size of `return address of the caller function` is fixed, for example, in 32 bits system the size is 4B. And `Base pointer` size is fixed as well, it's also 4B in a 32 bits system. This fixed size is important, you'll see the reason in following sections. 

Next, in the above diagram you see two kinds of pointers: **base pointer** and **stack pointer**. Let's check them one by one.

***Stack pointer***: pointer to the top of the stack. When stack adds or removes data then stack pointer will change correspondingly. Stack pointer is straight forward and not difficult to understand, right?
***Base pointer***: is also called `frame pointer` which points to the current active  frame. And the current active frame is the topmost frame of the stack. 

The base pointer is conventionlly used to mark the start of a function's stack frame, and the area of the stack managed by that function. As we mentioned above, since the size of return address and base pointer is fixed, so based on the address of base pointer you can get all the data in that stack frame. Local variables can be accessed with positive offset and passed parameters can be got with negative offset. That's the reason why it is called **base** pointer. Great design, right? 

The other thing need to discuss is what't the content of base pointer part in each stack frame? In the above diagram you see that a 4 bytes data is pushed into the stack, we call it base pointer. But what's the data? In fact, base pointer is designed to `store the caller's base pointer address`. This is also a smart design to make function return works well. We'll discuss more about it later.

#### CPU register

To understand stack memory management, you'll need to know 3 interesting CPU register:

- **eip**: Instruction pointer register which stores the address of the next instruction to be run. 
- **esp**: Stack pointer register which stores the address of the top of the stack at any time.
- **ebp**: Base pointer register which stores base pointer address of callee's stack frame.  And the content at this address is the caller's base pointer value (we already mentioned this point above).

Until now, you see all the necessary stuff: stack frame, stack frame content and CPU registers. Let's see how they play together to make stack frames of function call and return works. You'll see how this simple but beautiful design realizes such a complex task.  

### Mechanism of function call and return 

In this section, you'll understand how function call and return works by reading a few assembly codes (which are not difficult to understand).

#### function call
**Step one**: as you already see in the above diagram the first part of each stack frame is the passed parameters to the callee. So all the arguments are pushed on the stack as the following codes shows:

```assembly
push arg2;
push arg1;
```

`push` is the assembly instruction to push data onto the stack. And usually the arguments are pushed to the stack in the reverse order that they're declared in function.

**Step two**: the second part is the `Return address of the caller fn`, so we need to push the address of next instruction in caller function as `Return address` in callee's stack frame. As we introduced in the last section, the address of next instruction will be stored in `EIP` register, right? The assembly code goes as following: 

```assembly
push eip;
```

**Step three**: upon entrying to the callee function, the old `EBP` value (the caller function's base pointer address) is pushed onto the stack and then `EBP` is set to the value of `ESP`. Then the `ESP` is decremented (since the stack grows downward in stack memory) to allocate space for the local variables. And the codes goes as following:

```assembly 
push ebp;
mov ebp esp;
```
`mov` - the mov instruction copies the data item referred to by its second operand into the location referred to by its first operand. 

So `mov %ebp %esp` just means set `EBP` a new value of `ESP`. Please note that, `ESP` value changes when data is pushed or popped onto/from the stack. But it's always points to the top of the stack. Before this `mov %ebp %esp` instruction, `ESP` is pointing to the address just after the `return address of the caller`, and it should be the address of callee's base pointer and just what `EBP` should store, this instruction makes sense, right?

From that on, during the execution of the callee function, the passed arguments to the function are located at the positive offsets from `EBP`, and the local variables are located at the negative offsets from `EBP`, you already see this conclusion above.

Inside a function, the stack would look like this: 

```
| <argument 2>       |
| <argument 1>       |
| <return address>   |
| <old ebp>          | <= %ebp
| <local var 1>      |
| <local var 2>      | <= %esp
```

#### function return
**Step one**: upon exit from the callee function, all the function has to do is set `ESP` to the value of `EBP`. In this way can simply deallocates/releases the local variables from the stack. And then it can also expose callee's base pointer on the top of the stack for next step operation.

```assembly
mov esp, ebp;
```

This instruction is restoring the saved value of `ESP` upon entering the function, at that point what we did is `mov ebp esp`. Smart design, right?

**Step two**: since `ESP` already reset to the address of base pointer, next step we can simply pop the old `EBP` value from the top of stack as following: 

```assembly
pop ebp;
```

`pop` instruction retrieves the topmost value from the stack and puts the value into the second operand, in this case is `EBP`. Remember the `callee` function's base pointer stores the `caller` function's base pointer, so this simple `pop ebp` instruction realize EBP register restore perfactly. Great design, right?

**Step three**: next is straight forward, we need to pop the caller function return address to `EIP`.

```assembly
pop eip;
```

Similar to step two, right? Now the system knows the next instruction (pointing to the return address in the caller function) need to run. The execution context is giving back to the caller function. 

Upon returning back to the caller function, it can then increase `ESP` register again to remove the passed function arguments it pushed onto the stack. At this point, the stack frames becomes the same as it was in prior to invoking the callee function. 


