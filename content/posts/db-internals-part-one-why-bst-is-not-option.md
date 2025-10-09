---
author: "Farbod Ahmadian"
title: "Database Internals (Part One): Why BST is not an option for storage engines?"
date: "2025-10-08"
tags: 
- database
- storage-engine
---

Databases and storage engines are one of my interesting topics to follow
and recently I started reading [Database Internals](https://www.databass.dev/) while implementing
one called [VeryLightSQL (VLsql)](https://github.com/farbodahm/verylightsql)
following steps from [Let's Build a Simple Database](https://cstack.github.io/db_tutorial/).

So in a blog post series I'm going to capture the important things that I find during my implementations;
Starting the first one with this idea that why we have so many many different tree data structures and
why each storage engine prefers one over the other one.


### BST for storage engine?

[Binary Search Trees (BST)](https://opendatastructures.org/ods-python/6_2_BinarySearchTree_Unbala.html)
is a sorted *in-memory* data structures that can be used with *O(log(n))* **on-average** for most of the
delete, insert and search operations.

In a worst case scenario a BST can become *pathological* which basically means it does not differ from
a linked list with an O(n) search case.

![pathological](/images/pathological-bst.png)


How ever we are talking about storage engines; Which they **persist** and worst case scenarios are important. 


#### Why BST is not an option for storage engines?

It's not just about these 2 and we can compile a list depending on the storage engine's use case
and access patterns, but let's discuss about:

1. **Locality**: As elements are inserted randomly in a BST, they are not guranteed to
be stored in contiguous memory locations (As there is no gurantee that a node is close to its parent or children).
This can lead to cache misses and increased disk I/O when accessing elements,
which is a significant performance concern for databases that rely on fast access to data.
So node child pointers may span across several disk pages.
    - There is another tree called Paged Binary Tree which tries to address this issue; So you 
that a new tree is introduced.
1. **Tree height** and **Fanout**: BSTs can become unbalanced, leading to increased tree height and
    decreased *fanout* (number of children per node). So as we have O(lg(n)) time complexity for search,
    insert and delete operations, the constant factors can be high due to increased tree height.
    - So now again we have other trees like AVL, Red-Black and many more that tries to address this issue.


### So, why there are so many tree data structures?

As we discussed above, there are different aspects to consider when choosing a tree data structure for a storage engine.
Different trees are optimized for different use cases and access patterns.
In the example above, we had to introduce 2 new trees to address the issues with BSTs.

* High fanout to improve the locality of neighboring keys.
* Low height to reduce the number of disk accesses required to find a key.

Now consider many more factors that affect the choice of tree data structure for a storage engine,
like concurrency, write patterns, read patterns, etc.
