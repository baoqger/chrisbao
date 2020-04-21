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


###  what is memoization?

Memoization is a general concept in computer science, Wikipedia explains it as following: 

> In computing, memoization or memoisation is an optimization technique used primarily to speed up computer programs by storing the results of expensive function calls and returning the cached result when the same inputs occur again.

Memoization is a great optimization solution of pure function. Generally speaking, `A pure function is a function where the return value is only determined by its input values, without side effects`. As you may know `Selector` is a pure function. 

`memoizedSelector` is just an ordinary selector function with memoization optimization. Next let's see how it works in the design of NgRx library.

### usage of memoizedSelector

### source code of memoizedSelector 

### explore the memoizedSelector method
