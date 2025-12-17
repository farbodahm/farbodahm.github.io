---
author: "Farbod Ahmadian"
title: "Database Internals (Part One): Why BST is not an option for storage engines?"
date: "2025-10-08"
description: "Why Binary Search Trees don't work for databases: exploring locality, fanout, and tree height problems in storage engines."
tags:
- database
- storage-engine
---

Recently I started reading [Database Internals](https://www.databass.dev/) while implementing
one called [VeryLightSQL (VLsql)](https://github.com/farbodahm/verylightsql).
I'm following steps from [Let's Build a Simple Database](https://cstack.github.io/db_tutorial/).

In this blog post series, I'm going to capture the important things I find during my implementations.
Let's start with this question: why do we have so many different tree data structures?
And why does each storage engine prefer one over another?


### BST for storage engine?

[Binary Search Trees (BST)](https://opendatastructures.org/ods-python/6_2_BinarySearchTree_Unbala.html)
is a sorted *in-memory* data structure with *O(log(n))* **on-average** time complexity for delete, insert, and search operations.

In a worst case scenario, a BST can become *pathological*.
This means it degrades into something like a linked list with O(n) search time.

![pathological](/images/pathological-bst.png)


However, we are talking about storage engines.
Storage engines need to **persist** data, and worst case scenarios matter a lot. 


#### Why BST is not an option for storage engines?

It's not just these two issues.
We can compile a longer list depending on the storage engine's use case and access patterns.
But let's discuss:

1. **Locality**: As elements are inserted randomly in a BST, they are not guaranteed to be stored in contiguous memory locations.
There is no guarantee that a node is close to its parent or children.
This can lead to cache misses and increased disk I/O when accessing elements.
For databases that rely on fast access to data, this is a big performance problem.
Node child pointers may span across several disk pages.
    - There is another tree called Paged Binary Tree which tries to address this issue.
2. **Tree height** and **Fanout**: BSTs can become unbalanced, leading to increased tree height and decreased *fanout* (number of children per node).
Even though we have O(lg(n)) time complexity for search, insert, and delete operations,
height of the tree can easily grow to affect the performance of these operations.
For example, in a pathological case, the height of the tree can become O(n).
    - Trees like AVL, Red-Black, and many more try to address this issue.

Also keep note that Tree height and Fanout are intimately related. Having a high
fanout can help keep the tree height low and increase the locality of neighboring keys.
In BSTs, the fanout is always 2, which is not optimal for storage engines.


### So, why there are so many tree data structures?

There are different aspects to consider when choosing a tree data structure for a storage engine.
Different trees are optimized for different use cases and access patterns.
In the examples above, we introduced 2 new trees to just address the 2 discussed issues with BSTs.

* High fanout to improve the locality of neighboring keys.
* Low height to reduce the number of disk accesses required to find a key.

There are many more factors that affect the choice of tree data structure.
Things like concurrency, write patterns, and read patterns all play a role.
