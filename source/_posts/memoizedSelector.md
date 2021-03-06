---
title: Understand NgRx memoizedSelector in source code level
date: 2020-04-20 20:15:50
tags:
---

### Background

`Selector` is an essential part of the entire NgRx state management system, which is much more complicated than the `action` and `reducer` part based on my learning and development experience. Sometimes I feel it is like a black box, which hides many excellent designs and techniques. I spend some time and dig into the source code of NgRx to take a look at the internals of the black box. This post (and later posts) will share some interesting points I found during the process. 

When using NgRx, developers always do something like this: 

``` javascript 
export const animalListState: MemoizedSelector<State, Animal[]> = createSelector(
  rootState,
  (state: RootState): AnimalListState => state.animalList
);
```

`createSelector` method return a selector function, which can be passed into the `store.select()` method to get the desired state out of the store. 

By default, the type of the returned function from `createSelector` is `MemoizedSelector<State, Result>`. Have you ever notice that? This post will introduce what it is and how it works. 


###  What is memoization?

Memoization is a general concept in computer science. Wikipedia explains it as follows: 

> In computing, memoization or memoisation is an optimization technique used primarily to speed up computer programs by storing the results of expensive function calls and returning the cached result when the same inputs occur again. 

You can find [many articles](https://dev.to/carlillo/understanding-javascripttypescript-memoization-o7k) online for explaining memoization with code examples. Simply speaking, a hash map is used for the cached result. Technically it's not difficult at all. 

Memoization is a great optimization solution of pure function. Generally speaking, `A pure function is a function where the return value is only determined by its input values, without side effects`.

As you may know, `Selector` is a pure function. `memoizedSelector` is just a normal selector function with memoization optimization. Next, let's see how it works in the design of the NgRx library.

### Source code of memoizedSelector 

In the source code of [`NgRx`](https://github.com/ngrx/platform/blob/master/modules/store/src/selector.ts), you can find the `selector` related code in the path of `platform/modules/store/src/selector.ts`.  

`selector.ts` file is roughly 700 lines, which hold all the functionalities of it. There are many interesting points inside this module, which I can share in another article, but this post focus on memoization. So I pick up all the necessary code and put it as follows: 

``` typescript 
export type AnyFn = (...args: any[]) => any;

export type ComparatorFn = (a: any, b: any) => boolean;

export type MemoizedProjection = {
    memoized: AnyFn;
    reset: () => void;
    setResult: (result?: any) => void;
};

export function isEqualCheck(a: any, b: any): boolean {
    return a === b;
};

function isArgumentsChanged(
    args: IArguments,
    lastArguments:IArguments,
    comparator: ComparatorFn
) {
    for (let i = 0; i < args.length ; i++) {
        if (!comparator(args[i], lastArguments[i])) {
            return true;
        }
    }
    return false;
}

export function defaultMemoize(
    projectionFn: AnyFn,
    isArgumentsEuqal = isEqualCheck,
    isResultEqual = isEqualCheck
): MemoizedProjection {
    let lastArguments: null | IArguments = null;
    let lastResult: any = null;
    let overrideResult: any;

    function reset() {
        lastArguments = null;
        lastResult = null;
    }

    function setResult(result: any = undefined) {
        overrideResult = result;
    }

    function memoized(): any {
        if (overrideResult !== undefined) {
            return overrideResult;
        } 
        if (!lastArguments) {
            lastResult = projectionFn.apply(null, arguments as any);
            lastArguments = arguments;
            return lastResult;
        }

        if (!isArgumentsChanged(arguments, lastArguments, isArgumentsEuqal)) {
            return lastResult;
        }

        const newResult = projectionFn.apply(null, arguments as any);
        lastArguments = arguments;

        if (isResultEqual(lastResult, newResult)) {
            return lastResult;
        }

        lastResult = newResult;
        return newResult;
    }
    return { memoized, reset, setResult};
}
```
There are many interesting Typescript stuff in the above code block. But for memoization, you can focus on the method `defaultMemoize`. In the following section, I will show you how it can make your program run faster. 

### Explore the memoizedSelector method

To show how memoization works, I create a simple method `slowFunction` as following, to simulate that a method running very slowly:  

``` typescript 
export function slowFunction(val: number): number {
    const pre = new Date();
    while (true) {
        const now = new Date();
        if (now.valueOf() - pre.valueOf() > 2000) {
            break;
        }
    }
    return val;
}
```
And then test it with the following scripts: 

``` typescript 
import { defaultMemoize } from "./memoizedSelector";
import { slowFunction } from "./slowFunction";

// run slowFunction without memoization
console.log("First call of slowFunction(2)");
let pre = new Date();
slowFunction(2);
console.log("It takes" + ((new Date()).valueOf() - pre.valueOf())/1000 +  "seconds \n");
console.log("Second call of slowFunction(2)");
pre = new Date();
slowFunction(2);
console.log("It takes" + ((new Date()).valueOf() - pre.valueOf())/1000 +  "seconds \n");

// run slowFunction with memoization

const fastFunction = defaultMemoize(slowFunction);
console.log("First call of fastFunction(2)");
pre = new Date();
fastFunction.memoized(2);
console.log("It takes" + ((new Date()).valueOf() - pre.valueOf())/1000 +  "seconds \n");
console.log("Second call of fastFunction(2)");
pre = new Date();
fastFunction.memoized(2);
console.log("It takes" + ((new Date()).valueOf() - pre.valueOf())/1000 +  "seconds \n");
```
The output goes as folllowing:

```
$ ts-node index.ts
First call of slowFunction(2)
It takes2.001seconds

Second call of slowFunction(2)
It takes2.001seconds

First call of fastFunction(2)
It takes2.002seconds

Second call of fastFunction(2)
It takes0.001seconds
```

Compared with the original `slowFunction` method, the memoized method `fastFunction` can directly output the result for the same input. That's the power of memoization, hope you can master it. 


