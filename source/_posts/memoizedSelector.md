---
title: Understand NgRx memoizedSelector in source code level
date: 2020-04-20 20:15:50
tags:
---

### Background

`Selector` is a very important part of the entire NgRx state management system, which is much more complex than the `action` and `reducer` part based on my learning and usage experience. Sometimes I feel it is like a black box, which hides many good design and techniques. I spend some time and dig into the source code of NgRx to take a look at the internals of the black box. This post (and later posts) will share some interesting points I found during the process. 

When using NgRx, developers always do something like this: 

``` javascript 
export const animalListState: MemoizedSelector<State, Animal[]> = createSelector(
  rootState,
  (state: RootState): AnimalListState => state.animalList
);
```

`createSelector` method return a selector function, which can be passed into the `store.select()` method to get the desired state out of the store. 



###  what is memoization?

### usage of memoizedSelector

### source code of memoizedSelector 

### explore the memoizedSelector method