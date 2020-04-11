---
title: algo-part2
date: 2020-04-11 21:00:20
tags:
---

## Part 2

### Circular List

Judge a linked list is circular or not

``` javascript
function circular(list) {
  let slow = list.getFirst();
  let fast = list.getFirst();

  while (fast.next && fast.next.next) {
    slow = slow.next;
    fast = fast.next.next;

    if (slow === fast) {
      return true;
    }
  }

  return false;
}
```