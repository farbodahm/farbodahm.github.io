---
author: "Farbod Ahmadian"
title: "Database Internals (Part Three): B-Tree Techniques You Should Know"
date: "2026-05-07"
description: "A look at practical techniques used in B-Trees and other tree structures: sibling links, overflow pages, rebalancing, compression, and more."
tags:
- database
- storage-engine
- algorithms
---

In [Part One](/posts/db-internals-part-one-why-bst-is-not-option), I talked about why there are so many tree data structures and what trade-offs each one makes. Different trees optimize for different things: fanout, height, write patterns, concurrency, and so on.

But here's the thing. Many of these trees share a common set of building blocks. Think of each one as a *technique*: a tool you can pick up and apply to whatever tree structure you're working with, depending on the problem you're trying to solve. None of these are exclusive to B-Trees, even though that's where I first ran into most of them.

The ideas here come mostly from Chapter 4 ("Implementing B-Trees") of the [Database Internals](https://www.databass.dev/) book. The chapter covers how real implementations deal with the messy parts of B-Trees: managing large numbers of pointers, handling splits and merges, cleaning up dead space, and keeping things efficient under heavy writes.


## Sibling Links

In a standard B-Tree, if you want to move from one leaf to the next, you have to go back up to the parent and then come back down. That's fine for single key lookups, but for range scans it's wasteful. Every extra traversal means more disk reads.

Sibling links fix this by connecting leaf nodes directly. Each leaf holds a pointer to its right neighbor (and sometimes its left neighbor too). Once you find the starting key of your range, you just follow the chain forward without touching any internal nodes.

![B+Tree with sibling pointers between leaf nodes](/images/btree-sibling-links.svg)

A range scan from `14` to `28` finds its starting position in `Leaf B`, then follows the dashed sibling chain into `Leaf C`, never going back up to the root.

This is actually what turns a B-Tree into a B+Tree. Most database implementations use this. PostgreSQL, MySQL's InnoDB, SQLite, they all keep forward pointers in their leaf nodes. I also added this to [VeryLightSQL](https://github.com/farbodahm/verylightsql) early on, since range queries would be painful without it.

![Range scan walks the sibling chain from the starting key until it passes the upper bound](/images/btree-range-scan.svg)

The scan just walks the linked list. No parent lookups, no re-traversals.


## Rightmost Pointer

In a B-Tree internal node, you store keys and child pointers. The typical layout is: pointer, key, pointer, key, pointer. If you have N keys, you have N+1 pointers. That extra pointer on the right side points to the subtree containing all keys greater than the last separator key.

Some implementations separate this rightmost pointer from the rest and store it in its own field. This simplifies the logic for searching and splitting because the key-pointer pairs become a clean array, and the "everything bigger than my last key" case is handled separately.

![Internal node with a key/pointer array and a separate rightmost pointer](/images/btree-rightmost-pointer.svg)

It's a small design choice, but it makes the code easier to reason about when you're doing splits and inserts on internal nodes. The search becomes a simple loop through the key-pointer pairs with a fallback to the rightmost child. Without that separation, you'd need extra index arithmetic to handle the last child.


## Overflow Pages

B-Tree nodes have a fixed size, usually matching the disk page size (4KB or 8KB). But what happens when a value is too large to fit in a single node? Think of a row with a big text column or a blob.

Instead of forcing a split or rejecting the insert, the node can store the value on a separate overflow page. The node itself just keeps a pointer to that page. The main page stays a predictable size, and the oversized data lives elsewhere.

![Leaf page with one row pointing to a separate overflow page that holds the large value](/images/btree-overflow-pages.svg)

This keeps the tree structure clean. Node splits and merges only deal with fixed-size pages. The overflow pages are managed separately, almost like a side heap.

PostgreSQL calls this [TOAST](https://www.postgresql.org/docs/current/storage-toast.html) (The Oversized-Attribute Storage Technique). When a row exceeds a size threshold, large columns get compressed and moved to a TOAST table, and the main tuple just stores a pointer.


## Breadcrumbs

When you traverse a B-Tree from root to leaf, you visit a sequence of internal nodes. If you save that path as you go down, you get a trail of breadcrumbs back to the root.

Why is this useful? After you insert a key into a leaf and it overflows, you need to split the leaf and push a key up to the parent. But how do you find the parent? You could store parent pointers in each node, but that means extra storage and extra writes every time nodes move around. Breadcrumbs give you the same information without storing anything in the tree itself.

You just keep a **stack** during traversal. After the insert, you pop from the stack to walk back up and handle splits at each level.
The breadcrumbs list is the path. Each split pops one level up. No parent pointers needed.

![Tree traversal with the path nodes pushed onto a breadcrumb stack](/images/btree-breadcrumbs.svg)


## Rebalancing

When you delete keys from a B-Tree, nodes can become underfull. The basic approach is to just merge underfull nodes with a sibling. But merging can cascade upward and cause more restructuring than necessary.

Rebalancing is a less aggressive alternative. Instead of immediately merging, you check if a sibling has extra keys to spare. If it does, you rotate a key from the sibling through the parent into the underfull node. This keeps both nodes valid without changing the tree's structure.

You only merge when neither sibling can spare a key. This reduces the number of structural changes and can cut down on disk writes.

![Rotation: a separator key drops from the parent into the underfull child, and the sibling's first key lifts up to take its place](/images/btree-rebalance.svg)

The rotation keeps both nodes above the minimum key count. Merging is the last resort.


## Write-Only Appends (Copy-on-Write)

Updating a node in place means you need to write to a specific location on disk. If the system crashes mid-write, that page is corrupted and you've lost both the old and the new version.

Write-only append (or copy-on-write) avoids this by never modifying existing pages. When you need to update a node, you write a new copy of it to the end of the file. The parent then gets updated (also via a new copy) to point to the new child. This cascades up to the root, which gets atomically swapped.

![Copy-on-write update: the modified leaf, its parent, and the root all get new copies; sibling subtrees are shared between v1 and v2](/images/btree-copy-on-write.svg)

The old pages become garbage after the new root is live. A background process can reclaim them later.

This approach gives you crash safety almost for free. If the system dies before the root pointer is updated, the old tree is still valid. The old page is never touched, so nothing gets corrupted. LMDB and Btrfs (the file system) both use this strategy. It also makes snapshots cheap because old roots remain readable as long as you don't reclaim their pages.


## Compression

B-Tree nodes often contain keys that share common prefixes. Think of keys like `user:1001`, `user:1002`, `user:1003`. Storing the full key every time wastes space.

Prefix compression (also called prefix truncation) stores the shared prefix once and then only records the differing suffix for each subsequent key. This lets you fit more keys per node, which increases fanout and reduces tree height.

Another form is page-level compression, where the entire page gets compressed before being written to disk. This trades CPU time for reduced I/O, which is usually a good deal since disk reads are orders of magnitude slower than decompression.

WiredTiger (MongoDB's storage engine) supports both prefix compression for keys and block compression (using snappy or zlib) for pages.

![Prefix compression: each key stores only the suffix that differs from the previous one](/images/btree-prefix-compression.svg)

The compressed form is much smaller, especially when keys are clustered, which they usually are in sorted trees.


## Wrapping Up

None of these techniques exist in isolation. A real B-Tree implementation will use several of them together. You might have sibling links for range scans, overflow pages for large values, breadcrumbs for traversal, and prefix compression to squeeze more keys per node. The combination depends on your workload and what problems you're actually hitting.

The main takeaway: these are *techniques*, not rules. They're tools sitting on a shelf. You pick the ones that solve the problems you have and leave the rest. Different databases make different choices here, and that's fine. Understanding what each technique does and when it helps puts you in a much better position to reason about why a particular database behaves the way it does.
