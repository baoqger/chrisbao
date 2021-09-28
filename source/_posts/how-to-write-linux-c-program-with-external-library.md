---
title: How to use libraries in Linux C program 
date: 2019-08-25 16:08:36
tags:
---

### Background

After you learned and understood how to write small C programs in Linux (that means you know the syntax of C lauguage and how to compile your programs with `gcc`), gradually you will always want to have some adventures to write more powerful and larger programs. To be a greate software engineer, you need this passion to explore, right?

To write a larger program, you can't build everything from scratch. This means you need to import and use libraries developed by others in your program.  

To be honest, for new developers of Linux C programming, the whole handling of libraries can be a mystery, which requires knowledge about `gcc compiler`, `Linux system`, `GNU conventions` and so on. In this article, I will focus on this topic and piece all these knowledge together. After reading this post, I hope you can feel free and confident to add the shared libraries from the open source world to your program. 

**Note**: To illustrate the discussion more clearly, this article is based on this demo github [app](https://github.com/baoqger/handle-c-library-demo-linux). And my demo app is forked from Stephan Avenwedde's original [repo](https://opensource.com/article/20/6/linux-libraries). Thanks for that.  

The code logic of this demo app is very stragitforward, so I will not review the code line by line in this article, which is not the target of this post. 

### Static library vs Shared library

Generally speaking there are two types of library: `Static` and `Shared`. 
- **static library**: is simply a collection of ordinary object files; conventionally, static libraies end with the `.a` suffix. This collection is created using `ar` command. For example, in the demo app, we build static library like this:
  
```makefile
CFLAGS =-Wall -Werror

libmy_static.a: libmy_static_a.o libmy_static_b.o
	ar -rsv ./lib/static/libmy_static.a libmy_static_a.o libmy_static_b.o

libmy_static_a.o: libmy_static_a.c
	cc -c libmy_static_a.c -o libmy_static_a.o $(CFLAGS)

libmy_static_b.o: libmy_static_b.c
	cc -c libmy_static_b.c -o libmy_static_b.o $(CFLAGS)
```
In our case, the static library will be installed in the subdirectory `./lib/static` and named as `libmy_static.a`.   

When the application links against a static library, the library's code becomes part of the resulting executable. So you can run the application without any further setting or configuration. That's the biggest advantage of static library, you'll see how it works in next section. 

Static libraries aren't used as often as they once were, since `shared library` has more advantages. 

- **shared library**: end with `.so` which means **shared object**. The shared library is loaded into memory when the application starts. If multiple applications use the same shared library, the shared library will be loaded only once. In our demo app, the shared library is build as follows: 

```makefile
libmy_shared.so: libmy_shared.o
	cc -shared -o ./lib/shared/libmy_shared.so libmy_shared.o

libmy_shared.o: libmy_shared.c
	cc -c -fPIC libmy_shared.c -o libmy_shared.o
```

The shared library `libmy_shared.so` is installed inside `./lib/shared` subdirectory. Note that the shared library is build with some special option for `gcc`, like `-fPIC` and `-shared`. In detail, you can check the document of gcc. 



### Naming convention

Generally speaking, the shared library follows the pattern : `lib` + name of library + `.so`. Of course, version information is very important, but for simplicity reason, in this article let's ingore the version issue.

### Link library

After building the libraries, next step you need to import the libraries in your app. For our app, the source code of the demo is the file of `demo/main.c` as follows:

```c
#include <stdio.h>
#include <libmy_shared.h>
#include <libmy_static_a.h>
#include <libmy_static_b.h>

int main(){

    printf("Press Enter to repeat\n\n");
    do{
        int n = getRandInt();
        n = negateIfOdd(n);
        printInteger(&n);

    } while (getchar() == '\n');
   
    return 0;
}
```

Note that we import three `header file` of our libraries. Next question is how to build the demo app. 

Let's recall the building process of a C program as follows:

<img src="/images/compile_process.png" title="compile process" width="800px" height="600px">

this is a two-step process: `compile` and `link`. Firstly, the source code file of `main.c` will be compiled to an object file, let's say `main.o` file. In this step, the critical thing is telling `gcc` where to find the `header file` of the libraries. Next, link the `main.o` object file with our libraries: `libmy_static.a` and `libmy_shared.so`. `gcc` needs to know the name of library it should link and where to find these libraries, right? 

### Filesystem placement convention

As I shown above, in our case 

build of main.c
  -l option order issue

where should we install shared library
    ldd
    ldconfig
    ld.so

