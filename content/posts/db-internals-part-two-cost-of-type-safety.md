---
author: "Farbod Ahmadian"
title: "Database Internals (Part Two): The Real Cost of Type Safety in Storage Engines"
date: "2026-01-18"
description: "Exploring the trade-off between unsafe pointer arithmetic and type-safe serialization in B-Tree storage engines, with real benchmark numbers."
tags:
- database
- storage-engine
- golang
- performance
---

In [Part One](/posts/db-internals-part-one-why-bst-is-not-option), I explored why Binary Search Trees aren't suitable for storage engines (primarily due to locality and fanout limitations).
Today, I want to share something I learned the hard way: **how you serialize your B-Tree nodes to disk can make or break your database performance**.

This isn't a theoretical discussion. As I kept adding features to [VeryLightSQL](https://github.com/farbodahm/verylightsql) (leaf node splitting, internal node splitting, parent pointer updates, initiating transactions)
the code was becoming increasingly difficult to maintain. Every new feature required careful byte offset calculations, and debugging meant staring at hex dumps.

* For a learning project where understanding the code matters more than raw performance, I decided to refactor toward something more readable.

I made a change that seemed like the "right" engineering decision, and it made the database **357x slower**. Let me walk you through what happened.

## The Original Approach: Living Dangerously with `unsafe`

When I first implemented VeryLightSQL, I followed the same approach SQLite uses: direct byte manipulation with pointer arithmetic. Here's what the code looked like:

```go
func leafNodeKey(node []byte, cellNum uint32) *uint32 {
    cell := leafNodeCell(node, cellNum)
    return (*uint32)(unsafe.Pointer(&cell[LeafNodeKeyOffset]))
}

func leafNodeNumCells(node []byte) *uint32 {
    return (*uint32)(unsafe.Pointer(&node[LeafNodeNumCellsOffset]))
}
```

The node is just a raw byte slice (`[]byte`), and we calculate exact byte offsets to read and write fields directly. No serialization, no deserialization; just raw memory access.

**Pros:**
- Blazing fast: zero serialization overhead
- Memory efficient: no intermediate copies
- Follows how real databases (SQLite, PostgreSQL) do it

**Cons:**
- Uses Go's `unsafe` package (the name is a warning)
- Manual offset calculations are error-prone
- Hard to debug as you're staring at byte offsets

## The Refactor: Proper Go Structs

As a learning project, I had a TODO in the code: "Use proper Go structs with serialization instead of byte arrays and unsafe." So I decided to do it:

```go
type LeafCell struct {
    Key   uint32
    Value Row
}

type LeafNode struct {
    NodeType uint8
    IsRoot   uint8
    Parent   uint32
    NumCells uint32
    NextLeaf uint32
    Cells    [LeafNodeMaxCells]LeafCell
}

func (n *LeafNode) Serialize() ([]byte, error) {
    buf := new(bytes.Buffer)
    if err := binary.Write(buf, binary.LittleEndian, n); err != nil {
        return nil, err
    }
    page := make([]byte, pageSize)
    copy(page, buf.Bytes())
    return page, nil
}
```

Beautiful, right? Type-safe, no `unsafe` package, proper Go idioms. The compiler catches errors, debugging is easy, and the code is readable.

## The Benchmark Reality Check

Then [I ran the benchmarks](https://github.com/farbodahm/verylightsql/pull/16):

| Operation | Before (unsafe) | After (encoding/binary) | Regression |
|-----------|-----------------|------------------------|------------|
| Sequential Insert | 51,441 ns/op | 18,376,824 ns/op | **357x slower** |
| Random Insert | 63,497 ns/op | 17,291,198 ns/op | **272x slower** |
| Cursor Advance (50 rows) | 253 ns/op | 2,037,652 ns/op | **8,044x slower** |
| FindKey (50 rows) | 31 ns/op | 20,536 ns/op | **660x slower** |

I stared at these numbers for a while. Over **8,000x slower** for cursor operations? That's not a minor regression, that's a different category of performance entirely! :)

## Why Is `encoding/binary` So Slow?

The `encoding/binary` package uses reflection to figure out struct layouts at runtime. Every time you call `binary.Read` or `binary.Write`, it:

1. Reflects over the struct to find field types and sizes
2. Allocates buffers for the serialization
3. Copies data through multiple layers
4. Handles byte ordering conversions

For a hot path like B-Tree node access (which happens on every single read and write operation) this overhead is unacceptable.

Here's the serialization benchmark comparison:

```
BenchmarkNodeSerialization/LeafNode_Serialize    5,875    20,600 ns/op
BenchmarkNodeSerialization/LeafNode_Deserialize  5,828    19,820 ns/op
```

Every node access now costs ~20 microseconds just for serialization. In a database where you might access dozens of nodes per query, this adds up fast.

## The Trade-off Triangle

This experience was an example for me about storage engine design. You're always balancing three concerns:

```
        Performance
            /\
           /  \
          /    \
         /      \
        /________\
   Safety      Simplicity
```

**Performance** (unsafe pointers): Direct memory access, zero overhead, but dangerous and complex.

**Safety** (`encoding/binary`): Type-safe, compiler-checked, but significant runtime overhead.

**Simplicity** (JSON/Gob): Easy to use, flexible schemas, but even slower and larger storage footprint.

Real databases choose performance. SQLite, PostgreSQL, MySQL they all use direct memory manipulation for their page structures.
They accept the complexity because the performance cost of "clean" approaches is simply too high.

## What I Learned

1. **Benchmarks aren't optional**: I would have shipped a 357x slower database if I hadn't had benchmarks in place. The code worked perfectly; it was just unusably slow.

You can read mre about it in [Go Application Benchmarking with GitHub PR Integration](/posts/benchmarking-golang)

2. **"Best practices" have context**: Using `unsafe` is generally discouraged in Go. But for a storage engine's hot path, it might be the right choice.

The Go standard library itself uses `unsafe` extensively for performance-critical code.

3. **Learning projects should break things**: I'm glad I tried the "clean" approach. Now I understand *why* databases use pointer arithmetic, not just that they do.

4. **Reflection is expensive**: Any time you're on a hot path, avoid reflection-based serialization. This applies to JSON, Gob, and `encoding/binary` with structs.

For VeryLightSQL, I'm keeping the slow version for now; it's a learning project, and the code clarity helps me understand what's happening. But if I were building a production database, I'd either use `unsafe` or write custom serialization methods.


*The full benchmark comparison is available in [PR #16](https://github.com/farbodahm/verylightsql/pull/16) of VeryLightSQL.*
