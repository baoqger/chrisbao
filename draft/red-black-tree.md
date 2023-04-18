---
title: "Understand Red Black Tree: part two - insertion"
date: 2023-02-15 10:08:13
tags: Algorithm, Data structure, Tree, Red Black Tree
---

### Introduction

A fascinating and important balanced binary search tree is the red-black tree. Rudolf Bayer invented this important tree structure in 1972, about 10 years after the introduction of AVL trees. He is also credited with the invention of the B-tree, a structure used extensively in database systems. Bayer referred to his red-black trees as “symmetric binary B-trees.”

A red-black tree satisfies the following properties:

- Red/Black Property: Every node is colored, either red or black.
- Root Property: The root is black.
- Leaf Property: Every leaf (NIL) is black.
- Red Property: If a red node has children then, the children are always black.
- Depth Property: For each node, any simple path from this node to any of its descendant leaf has the same black-depth (the number of black nodes).

It is rule 5 that leads to a balanced tree structure. Since red nodes may not have any red children, if the black height from root to every leaf node is the same, this implies that any two paths from root to leaf can differ only by a factor of two. Using a relatively simple proof of mathematical induction, Kenneth Berman and Jerome Paul show on page 659 of their book, Algorithms: Sequential, Parallel, and Distributed, Thomson, Course Technology, 2005, that the relationship between the maximum depth of a red-black tree and the number of internal nodes is, depth = 2 log2(n + 1) where n is the number of internal nodes.

The rules that define a red-black tree are interesting because they are less constrained than the rather strict rules associated with an AVL tree.

How the red-black tree maintains the property of self-balancing?

The red-black color is meant for balancing the tree.The limitations put on the node colors ensure that any simple path from the root to a leaf is not more than twice as long as any other such path. It helps in maintaining the self-balancing property of the red-black tree.