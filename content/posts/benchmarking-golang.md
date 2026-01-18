---
author: "Farbod Ahmadian"
title: "Go Application Benchmarking with GitHub PR Integration"
date: "2026-01-14"
description: "A practical guide to benchmarking Go applications, integrating benchmarks into GitHub PRs, and why benchmarks are one of the best learning tools for developers."
tags:
- golang
- benchmarking
- performance
---

Last week, I refactored the B-Tree serialization layer in [VeryLightSQL](/posts/db-internals-part-two-cost-of-type-safety), my toy database project. The new code was cleaner, more idiomatic, and eliminated all uses of Go's `unsafe` package.

After opening [the PR](https://github.com/farbodahm/verylightsql/pull/16),  GitHub Actions posted a comment on my PR:

```
Performance Alert
BenchmarkInsert/Sequential: 357.24x regression
BenchmarkCursor/Advance_50rows: 8,044.58x regression
```

A **dramatic performance regression**. The code worked perfectly and all tests passed.
But it would have made my database essentially unusable.

This is why I benchmark everything.

## Benchmarks Are a Learning Tool

I want to start with something that doesn't get said enough: **benchmarks teach you more about your code than almost anything else**.

When you write a benchmark, you're forced to think about:
- What operations are actually hot paths?
- How does performance scale with input size?
- Where are the hidden allocations?

When you run benchmarks regularly, you learn:
- Which abstractions have real costs
- How the Go runtime behaves under different conditions
- Why certain patterns exist in production code

I've learned more about Go's memory model, garbage collector behavior, and standard library internals from benchmarking than from any documentation. Numbers don't lie, they reveal what's actually happening underneath your abstractions.

## Setting Up Go Benchmarks

Go's built-in benchmarking is straightforward. Here's a basic example from [VeryLightSQL](https://github.com/farbodahm/verylightsql):

```go
func BenchmarkInsert(b *testing.B) {
    b.Run("Sequential", func(b *testing.B) {
        for range b.N {
            b.StopTimer()
            table, cleanup := setupBenchmarkTable(b)
            b.StartTimer()

            for j := range 100 {
                table.Insert(createRow(int32(j)))
            }

            b.StopTimer()
            cleanup()
        }
    })
}
```

Key patterns I follow:

1. **Use sub-benchmarks** (`b.Run`) to test variations
2. **Stop the timer** for setup/teardown operations
3. **Test realistic scenarios**, not micro-operations
4. **Include multiple sizes** to understand scaling behavior

Run benchmarks with:

```bash
go test -bench=. -benchmem
```

The `-benchmem` flag is crucial; it shows allocations per operation, which often matters more than raw speed.

## Posting Benchmark Results on GitHub PRs

The real power comes from automated benchmark comparison. I use [benchmark-action](https://github.com/benchmark-action/github-action-benchmark) in my GitHub Actions workflow:

```yaml
name: Benchmark

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Run benchmarks
        run: go test -bench=. -benchmem -run=^$ | tee benchmark-results.txt

      - name: Compare benchmarks
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'go'
          output-file-path: benchmark-results.txt
          github-token: ${{ secrets.GITHUB_TOKEN }}
          alert-threshold: '150%'
          comment-on-alert: true
          fail-on-alert: false
```

This setup:
1. Runs benchmarks on every PR
2. Compares against the `main` branch baseline
3. Posts a comment if any benchmark regresses by more than 50%
4. Shows the exact regression ratio for each affected benchmark

Here's what the alert looks like on a PR:

```markdown
⚠️ **Performance Alert** ⚠️

Possible performance regression was detected for benchmark.
Benchmark result of this commit is worse than the previous benchmark result exceeding threshold `1.50`.

| Benchmark | Previous | Current | Ratio |
|-----------|----------|---------|-------|
| BenchmarkInsert/Sequential | 51,441 ns/op | 18,376,824 ns/op | 357.24 |
```

This caught my drastic regression before it hit main. Without automated comparison, I might have assumed "the tests pass, ship it."

## Learning from Streame's Benchmarking Setup

I use a more sophisticated setup in [Streame](https://github.com/farbodahm/streame), my stream processing library. There, I generate CPU and memory profiles as SVG visualizations:

```bash
#!/bin/bash
# benchmarks/take_benchmarks.sh

BENCHMARKS=$(go test ./... -list Benchmark | grep Benchmark)

for BENCHMARK in $BENCHMARKS; do
    TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")

    # Run with CPU profiling
    go test ./benchmarks -bench="^${BENCHMARK}$" \
        -cpuprofile="${TIMESTAMP}_${BENCHMARK}_cpu.prof" \
        -benchtime=5s

    # Generate SVG visualization
    go tool pprof -svg "${TIMESTAMP}_${BENCHMARK}_cpu.prof" \
        > "${TIMESTAMP}_${BENCHMARK}_cpu.svg"
done
```

The SVG reports get uploaded as GitHub Actions artifacts, so I can visually inspect where time is being spent:

```yaml
- name: Upload benchmark reports
  uses: actions/upload-artifact@v4
  with:
    name: benchmark-reports
    path: benchmarks/*.svg
    retention-days: 30
```

This helps answer "why is it slow?" not just "is it slow?"

## My Benchmark Design Philosophy

After maintaining benchmarks across multiple projects, here are patterns that work:

### 1. Benchmark Real Operations, Not Micro-Operations

Bad:
```go
func BenchmarkMapAccess(b *testing.B) {
    m := make(map[string]int)
    m["key"] = 1
    for range b.N {
        _ = m["key"]
    }
}
```

Better:
```go
func BenchmarkFindKey(b *testing.B) {
    table := setupTableWith200Rows()
    b.ResetTimer()
    for i := range b.N {
        table.findKey(uint32(i % 200))
    }
}
```

The second benchmark tests what users actually do. Micro-benchmarks are useful for understanding primitives, but integration-level benchmarks catch real regressions.

### 2. Test Multiple Scales

```go
func BenchmarkSelectAll(b *testing.B) {
    for _, rowCount := range []int{50, 100, 200, 500} {
        b.Run(fmt.Sprintf("Rows_%d", rowCount), func(b *testing.B) {
            table := setupTableWithRows(rowCount)
            b.ResetTimer()
            for range b.N {
                _ = table.SelectAll()
            }
        })
    }
}
```

This reveals O(n) vs O(n²) behaviors that single-size benchmarks miss.

### 3. Include Allocation Metrics

Always run with `-benchmem`. A function might be fast but allocate heavily, causing GC pressure in production:

```
BenchmarkInsert/Sequential-10    10    10,587,679 ns/op    4,521,088 B/op    52,341 allocs/op
```

Those 52,341 allocations per operation will hurt in a long-running service.

Not unrelated to benchmarking and Go GC, Twich's blog on [Memory Ballast](https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/) worths a read.

### 4. Benchmark Your Hot Paths

Identify what operations happen most frequently. For a database:
- Key lookups (every query)
- Cursor advancement (every row read)
- Node serialization (every disk access)

For a web service:
- Request parsing
- Authentication checks
- Response serialization

Focus benchmark effort where performance matters most.

## The Value of Benchmarking

Setting up benchmarks takes time. Is it worth it?

For VeryLightSQL, the benchmark suite caught:
- A 357x regression in insert operations
- An dramatic regression in cursor operations
- Memory allocation increases of 37x

One of these shipping to production would have been a disaster. The benchmark setup took maybe 2-3 hours. The alternative [Ex. debugging performance issues in production or, worse, not noticing them] would cost far more.

More importantly, *benchmarks grow more valuable over time*. Once the setup is done, every future change is automatically checked for performance. The upfront work keeps paying off.

## Getting Started

If you're not benchmarking yet, start simple:

1. **Write one benchmark** for your most critical operation
2. **Run it locally** before and after changes
3. **Add it to CI** with basic comparison
4. **Expand coverage** as you identify more hot paths

You don't need a perfect setup from day one. Even manual `go test -bench=.` before merging catches obvious regressions.

The goal isn't perfect benchmarks; **it's building the habit of measuring performance**. Once you start seeing the numbers, you'll never want to ship without them.

*See the benchmark setup in action: [VeryLightSQL benchmarks](https://github.com/farbodahm/verylightsql/blob/main/benchmark_test.go) and [Streame benchmarks](https://github.com/farbodahm/streame/tree/main/benchmarks).*
