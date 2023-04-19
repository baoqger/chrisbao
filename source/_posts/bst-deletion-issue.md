---
title: "Deletion operation in Binary Search Tree: successor or predecessor"
date: 2023-01-01 12:10:04
tags: Binary Search Tree, delete, balanced, performance
---

### Introduction

Today, in this article I want to examine one concrete topic: how to delete a node from a `binary search tree`. This question is important for understanding other tree data structures, like the [AVL tree](https://en.wikipedia.org/wiki/AVL_tree), and the [Red-Black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree). 

As is common with many data structures, `deletion` is always the hardest operation. For example, to delete a specific element from the [array](https://en.wikipedia.org/wiki/Array), we need to shift all the elements after it by one index. And to delete one node from the [linked list](https://en.wikipedia.org/wiki/Linked_list), we need to reset the pointer of the previous node (if it's a doubled linked list, we need to reset more points, which is a more complex case). This is the same for the binary search tree as well. Let's see in the following section. 

### Deletion of the binary search tree

Before we can examine the deletion operation in depth, let's quickly review the concept of the binary search tree as follows:

- Binary search tree: is a binary tree holding the following property that for every node, X, in the tree, the values of all the keys in its left subtree are smaller than the key value in X, and the values of all the keys in its right subtree are larger than the key value in X. 

Simply speaking, the binary search tree is a binary tree satisfying the `binary search` property. So each node of a binary search tree can have two children subtrees at most. When we need to delete one node from the binary search tree, then it's necessary to consider the following 3 cases. 

If the node is a `leaf`, then it can be deleted immediately. For example, to delete node `1` from the following binary search tree: 

<img src="/images/left-node-deletion.png" title="Delete leaf node from binary search tree" width="400px" height="500px">

I will show you how to implement it at the code level later, which summarizes all the cases. 

If the node has only `one child subtree`, the node can be deleted after we reset its parent's pointer to bypass the node and point to its child node. For example, to delete node `4` as follows: 

<img src="/images/onechild-node-deletion.png" title="Delete leaf node from binary search tree" width="400px" height="500px">

In the above example, node `4` has only one left child node `3`. Let's consider the symmetric scenario, imagine what will happen when it has only one right child node. The answer is it doesn't influence how we handle the deleted node here and the result is the same. I will not draw the diagram here and leave it for you to explore. 

The complicated case is how to deal with a node with `two children`. Before we introduce the solution, let's clarify one concept about the binary search tree: `successor`: 

- Successor: is the node with the `minimum` value in the `right` subtree of any node.


<img src="/images/twochildren-delete-origin.png" title="Delete leaf node from binary search tree" width="200px" height="300px">

Let's take node `16` in the above binary search tree as an example, the `successor` is node `17`. All right! Based on this concept, the solution to delete a node with two children is straightforward:

- Replace the data of the node with the value of the `successor` and recursively delete the `successor` from the `right` subtree. 

Let's try to analyze this solution. Firstly, replacing the data of the node with the value of the `successor` can keep the `binary search` property after the deletion operation. Secondly, since the `successor` has the minimum value of the subtree, which means it cannot have a left child. So the `successor` is either a leaf node without any child node or a node with only one right child node. So recursively deleting the `successor` can be resolved by the two simple cases we discussed above, it's perfect, right?  For instance, deletion of node `16` goes as follows: 

<img src="/images/twochildren-delete-after.png" title="Delete leaf node from binary search tree" width="500px" height="600px">




show real code

讨论：两种方法：predecessor 可行吗？

延申：unbalanced，performance degradation, self-balanceing BST
