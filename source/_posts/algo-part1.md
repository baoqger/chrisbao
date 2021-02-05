---
title: algo-part1
date: 2020-04-10 21:00:20
tags:
---
The Coding Interview Bootcamp: Data structure and algorithm.

## Part 1

### String Reverse

``` javascript
function reverse(str) {
  return str.split('').reduce((rev, char) => char + rev, '');
}
```

### Paldinromes 

``` javascript
function palindrome(str) {
  return str.split('').every((char, i) => {
    return char === str[str.length - i - 1];
  });
}
```
another way is based on recursive method:

``` javascript
function palindromeRecursive(str) {
  if (str.length <= 1) {
    return true;
  } else {
    if (str[0] !== str[str.length - 1]) {
      return false;
    } else {
      return palindromeRecursive(str.substring(1, str.length - 1))
    }
  }
}
```

### Longest Paldinromes substring

``` javascript
function longestPalin(str) {
  str = treatStr(str);
  console.log(str)
  let max =  0;
  for (let i  = 1; i < str.length - 1; i++) {
    let radius = Math.min (i, str.length - i - 1);
    console.log(radius)
    let currentMax = getMaxSubLength(str, i, radius);
    if (max < currentMax) {
      max = currentMax + 1;
    }
  }
  return max;
}

function getMaxSubLength(str, i, radius) {
  let max = 0;
  for (let j = 0; j <= radius; j++) {
    if (str[i - j] === str[i + j]) {
      max = j;
    } else {
      break;
    }
  }
  return max;
}

function treatStr(str) {
  let rtn = '';
  for (let i = 0; i < str.length; i++) {
    if (i === str.length - 1) {
      rtn = rtn + str[i];
    } else  {
      rtn = rtn + str[i] + '#';
    }

  }
  return rtn;
}
```

### Integer Reversal

``` javascript
// --- Directions
// Given an integer, return an integer that is the reverse
// ordering of numbers.
// --- Examples
//   reverseInt(15) === 51
//   reverseInt(981) === 189
//   reverseInt(500) === 5
//   reverseInt(-15) === -51
//   reverseInt(-90) === -9

function reverseInt(n) {
  const reversed = n
    .toString()
    .split('')
    .reverse()
    .join('');

  return parseInt(reversed) * Math.sign(n);
}
```

### Anagrams

method1

``` javascript
function anagrams(stringA, stringB) {
  const aCharMap = buildCharMap(stringA);
  const bCharMap = buildCharMap(stringB);

  if (Object.keys(aCharMap).length !== Object.keys(bCharMap).length) {
    return false;
  }

  for (let char in aCharMap) {
    if (aCharMap[char] !== bCharMap[char]) {
      return false;
    }
  }

  return true;
}

function buildCharMap(str) {
  const charMap = {};

  for (let char of str.replace(/[^\w]/g, '').toLowerCase()) {
    charMap[char] = charMap[char] + 1 || 1;
  }

  return charMap;
}
```

method2

``` javascript
function anagrams(stringA, stringB) {
  return cleanString(stringA) === cleanString(stringB);
}

function cleanString(str) {
  return str
    .replace(/[^\w]/g, '')
    .toLowerCase()
    .split('')
    .sort()
    .join('');
}
```

### Printing Steps

The effect we need is as following: 

```
--- Examples
  steps(2)
      '# '
      '##'
  steps(3)
      '#  '
      '## '
      '###'
  steps(4)
      '#   '
      '##  '
      '### '
      '####'
```

solution based on recursive methods

``` javascript
function steps(n, row = 0, stair = '') {
  if (n === row) {
    return;
  }

  if (n === stair.length) {
    console.log(stair);
    return steps(n, row + 1);
  }

  const add = stair.length <= row ? '#' : ' ';
  steps(n, row, stair + add);
}
```

### Enter the Matrix Spiral

The requirement goes as following: 
```
--- Examples
  matrix(2)
    [[undefined, undefined],
    [undefined, undefined]]
  matrix(3)
    [[1, 2, 3],
    [8, 9, 4],
    [7, 6, 5]]
 matrix(4)
    [[1,   2,  3, 4],
    [12, 13, 14, 5],
    [11, 16, 15, 6],
    [10,  9,  8, 7]]
```

solution: 
``` javascript
function matrix(n) {
  const results = [];

  for (let i = 0; i < n; i++) {
    results.push([]);
  }

  let counter = 1;
  let startColumn = 0;
  let endColumn = n - 1;
  let startRow = 0;
  let endRow = n - 1;
  while (startColumn <= endColumn && startRow <= endRow) {
    // Top row
    for (let i = startColumn; i <= endColumn; i++) {
      results[startRow][i] = counter;
      counter++;
    }
    startRow++;

    // Right column
    for (let i = startRow; i <= endRow; i++) {
      results[i][endColumn] = counter;
      counter++;
    }
    endColumn--;

    // Bottom row
    for (let i = endColumn; i >= startColumn; i--) {
      results[endRow][i] = counter;
      counter++;
    }
    endRow--;

    // start column
    for (let i = endRow; i >= startRow; i--) {
      results[i][startColumn] = counter;
      counter++;
    }
    startColumn++;
  }

  return results;
}
```

### Fibonacci

solution based on **memoization** and **recursive**.

``` javascript
function memoize(fn) {
  const cache = {};
  return function(...args) {
    if (cache[args]) {
      return cache[args];
    }

    const result = fn.apply(this, args);
    cache[args] = result;

    return result;
  };
}

function slowFib(n) {
  if (n < 2) {
    return n;
  }

  return fib(n - 1) + fib(n - 2);
}

const fib = memoize(slowFib);
```

The most simplified method goes as following:

``` javascript
function memoize(fn) {
  const cache = {};
  return function(n) {
    if(cache[n]) {
      return cache[n];
    }
    const result = fn(n);
    cache[n] = result;

    return result;
  }
}
```

Make it more abstract like below:

``` javascript
function memoize(fn) {
  const cache = {};
  return function(...args) {
    if (cache[args]) {
      return cache[args];
    }

    const result = fn.apply(this, args);
    cache[args] = result;

    return result;
  };
}
```
### Queue and Stack

``` javascript
class Queue {
  constructor() {
    this.data = [];
  }

  add(record) {
    this.data.unshift(record);
  }

  remove() {
    return this.data.pop();
  }
}

class Stack {
  constructor() {
    this.data = [];
  }

  push(record) {
    this.data.push(record);
  }

  pop() {
    return this.data.pop();
  }

  peek() {
    return this.data[this.data.length - 1];
  }
}
``` 

### Two Become One 

Implement a Queue data structure using two stacks

``` javascript
const Stack = require('./stack');

class Queue {
  constructor() {
    this.first = new Stack();
    this.second = new Stack();
  }

  add(record) {
    this.first.push(record);
  }

  remove() {
    while (this.first.peek()) {
      this.second.push(this.first.pop());
    }

    const record = this.second.pop();

    while (this.second.peek()) {
      this.first.push(this.second.pop());
    }

    return record;
  }

  peek() {
    while (this.first.peek()) {
      this.second.push(this.first.pop());
    }

    const record = this.second.peek();

    while (this.second.peek()) {
      this.first.push(this.second.pop());
    }

    return record;
  }
}
```

### Linked List

``` javascript
class Node {
  constructor(data, next = null) {
    this.data = data;
    this.next = next;
  }
}

class LinkedList {
  constructor() {
    this.head = null;
  }

  insertFirst(data) {
    this.head = new Node(data, this.head);
  }

  size() {
    let counter = 0;
    let node = this.head;

    while (node) {
      counter++;
      node = node.next;
    }

    return counter;
  }

  getFirst() {
    return this.head;
  }

  getLast() {
    if (!this.head) {
      return null;
    }

    let node = this.head;
    while (node) {
      if (!node.next) {
        return node;
      }
      node = node.next;
    }
  }

  clear() {
    this.head = null;
  }

  removeFirst() {
    if (!this.head) {
      return;
    }

    this.head = this.head.next;
  }

  removeLast() {
    if (!this.head) {
      return;
    }

    if (!this.head.next) {
      this.head = null;
      return;
    }

    let previous = this.head;
    let node = this.head.next;
    while (node.next) {
      previous = node;
      node = node.next;
    }
    previous.next = null;
  }

  insertLast(data) {
    const last = this.getLast();

    if (last) {
      // There are some existing nodes in our chain
      last.next = new Node(data);
    } else {
      // The chain is empty!
      this.head = new Node(data);
    }
  }

  getAt(index) {
    let counter = 0;
    let node = this.head;
    while (node) {
      if (counter === index) {
        return node;
      }

      counter++;
      node = node.next;
    }
    return null;
  }

  removeAt(index) {
    if (!this.head) {
      return;
    }

    if (index === 0) {
      this.head = this.head.next;
      return;
    }

    const previous = this.getAt(index - 1);
    if (!previous || !previous.next) {
      return;
    }
    previous.next = previous.next.next;
  }

  insertAt(data, index) {
    if (!this.head) {
      this.head = new Node(data);
      return;
    }

    if (index === 0) {
      this.head = new Node(data, this.head);
      return;
    }

    const previous = this.getAt(index - 1) || this.getLast();
    const node = new Node(data, previous.next);
    previous.next = node;
  }

  forEach(fn) {
    let node = this.head;
    let counter = 0;
    while (node) {
      fn(node, counter);
      node = node.next;
      counter++;
    }
  }

  *[Symbol.iterator]() {
    let node = this.head;
    while (node) {
      yield node;
      node = node.next;
    }
  }
}
```

### Find the midpoint of linked list

``` javascript
--- Directions
Return the 'middle' node of a linked list.
If the list has an even number of elements, return
the node at the end of the first half of the list.
*Do not* use a counter variable, *do not* retrieve
the size of the list, and only iterate
through the list one time.
--- Example
  const l = new LinkedList();
  l.insertLast('a')
  l.insertLast('b')
  l.insertLast('c')
  midpoint(l); // returns { data: 'b' }

function midpoint(list) {
  let slow = list.getFirst();
  let fast = list.getFirst();

  while (fast.next && fast.next.next) {
    slow = slow.next;
    fast = fast.next.next;
  }

  return slow;
}
```

### Multiply two Big numbers 

``` javascript
function addbyStr(str1, str2) {
  let len = Math.max(str1.length, str2.length);
  let increase = 0;
  let rtn = '';
  for (let i = 0; i < len; i++) {
    let ope1 = i <= str1.length - 1 ? str1[str1.length - 1 -i] : '0';
    let ope2 = i <= str2.length - 1 ? str2[str2.length - 1 -i] : '0';
    let temp = parseInt(ope1) + parseInt(ope2) + increase;
    if (temp >= 10) {
      increase = 1;
      temp = temp - 10;
    } else {
      increase = 0;
    }
    rtn = temp + rtn;
  }
  return rtn;
}


function addArr(arr) {
  let result = arr.reduce((rtn, each) => {
    return rtn = addbyStr(rtn, each);
  });
  return result;
}

function mutliTwoStr(str1, str2) {
  let rtn = [];
  for (let i = 0; i < str2.length; i++) {
    let count = parseInt(str2[str2.length - 1 - i]);
    let temp = new Array(count).fill(str1)
    let tempResult = addArr(temp)
    tempResult =  tempResult + '0'.repeat(i)
    rtn.push(tempResult)
  }
  return addArr(rtn);
}
```