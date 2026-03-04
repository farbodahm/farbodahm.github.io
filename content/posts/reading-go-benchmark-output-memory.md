---
author: "Farbod Ahmadian"
title: "Reading Go Benchmark Output (Part One): Memory"
date: "2026-03-03"
description: "A practical guide to reading and understanding every field in Go benchmark output, with real examples from a concurrent sharded map implementation."
tags:
- golang
- benchmarking
- performance
- kata
---

A year ago when I started to learn about benchmarking in Go, I couldn't find an easy-to-follow guide that also included the small details.
Most resources either showed you how to *write* benchmarks or jumped straight into optimization. None of them really explained how to *read* the output line by line.

While implementing [Concurrent Map with Sharded Locks Go Kata](https://github.com/farbodahm/go-kata/pull/3/), I decided to write two blog posts on how to read Go profiling output in two parts: memory and CPU benchmarking.
All examples in this post come from that PR, a concurrent sharded map implementation.

If you want to learn about writing benchmarks and integrating them into CI, check out my earlier post on [Go Application Benchmarking with GitHub PR Integration](/posts/benchmarking-golang).
This post is about something different: understanding what the output *means*.

## The Code We're Benchmarking

The kata asks you to build a `ShardedMap`; a concurrent map that distributes keys across multiple shards to reduce lock contention.
The benchmark tests how `Set()` performs with 1 shard (heavy contention) vs 64 shards (distributed locks).

Here's the benchmark function:

```go
func benchmarkShardedSet(b *testing.B, shardCount int) {
    m := NewShardedMap[int, int](shardCount)
    numGoroutines := runtime.GOMAXPROCS(0)

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Set(i%10000, i)
            i++
        }
    })
    b.ReportMetric(float64(numGoroutines), "goroutines")
}

func BenchmarkSet_1Shard(b *testing.B) {
    benchmarkShardedSet(b, 1)
}

func BenchmarkSet_64Shards(b *testing.B) {
    benchmarkShardedSet(b, 64)
}
```

And a simpler sequential benchmark for `Get()`:

```go
func benchmarkShardedGet(b *testing.B, shardCount int) {
    m := NewShardedMap[int, int](shardCount)
    for i := 0; i < 10000; i++ {
        m.Set(i, i)
    }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Get(i % 10000)
            i++
        }
    })
}
```

Nothing fancy. Set some keys, get some keys, measure how fast it goes.

## Running the Benchmark

The command:

```bash
go test -bench=. -benchmem -count=3
```

Three flags:
- `-bench=.` matches all benchmark functions (the `.` is a regex)
- `-benchmem` adds memory allocation stats to the output
- `-count=3` runs each benchmark 3 times for consistency checking

Here's the output from my M1 Pro:

```
goos: darwin
goarch: arm64
pkg: github.com/farbodahm/go-kata/02-performance-allocation/02-concurrent-map-with-sharded-locks
cpu: Apple M1 Pro
BenchmarkSet_1Shard-10     5849028    208.0 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkSet_1Shard-10     5807408    207.8 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkSet_1Shard-10     5823493    208.7 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkSet_64Shards-10   20112658   61.54 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkSet_64Shards-10   19281560   62.06 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkSet_64Shards-10   19974573   61.56 ns/op    10.00 goroutines    0 B/op    0 allocs/op
BenchmarkGet_1Shard-10     9219829    145.6 ns/op    0 B/op    0 allocs/op
BenchmarkGet_1Shard-10     8037270    143.7 ns/op    0 B/op    0 allocs/op
BenchmarkGet_1Shard-10     8653832    141.8 ns/op    0 B/op    0 allocs/op
BenchmarkGet_64Shards-10   56095298   21.06 ns/op    0 B/op    0 allocs/op
BenchmarkGet_64Shards-10   54659326   21.18 ns/op    0 B/op    0 allocs/op
BenchmarkGet_64Shards-10   56462284   21.03 ns/op    0 B/op    0 allocs/op
PASS
ok  github.com/farbodahm/go-kata/...    20.523s
```

Let's break down every piece.

## The Header

```
goos: darwin
goarch: arm64
pkg: github.com/farbodahm/go-kata/02-performance-allocation/02-concurrent-map-with-sharded-locks
cpu: Apple M1 Pro
```

This is metadata for reproducibility.
`goos` and `goarch` are the OS and CPU architecture.
`cpu` is the specific processor.

Benchmark numbers are only meaningful on the same hardware.
If someone runs this on an `amd64` Linux machine, they'll get different numbers. The header makes it clear what produced these results.

## The Benchmark Name

Look at: `BenchmarkSet_1Shard-10`

This has two parts:
- **`BenchmarkSet_1Shard`**: the function name, exactly as you wrote it in Go
- **`-10`**: the value of `GOMAXPROCS`, meaning Go used 10 OS threads for parallelism

My M1 Pro has 10 cores (8 performance + 2 efficiency), so `GOMAXPROCS` defaults to 10.
If you run this on a 4-core machine, you'd see `-4` instead.

This matters because `b.RunParallel` distributes work across `GOMAXPROCS` goroutines.
The `-10` tells you exactly how many goroutines were competing for locks in this benchmark.

## Iterations

```
BenchmarkSet_1Shard-10     5849028    208.0 ns/op
BenchmarkSet_64Shards-10   20112658   61.54 ns/op
```

The first number (`5849028`, `20112658`) is the **iteration count**: how many times the operation ran.

You don't choose this number. Go's test framework auto-tunes it.
It starts small (1, 100, 10000, ...) and keeps increasing until the total runtime is stable (about 1 second).

A higher iteration count means the operation is faster. The framework could fit more iterations into the time budget.
Notice: 64 shards got **20 million iterations** vs 1 shard's **5.8 million**. That alone tells you 64 shards is significantly faster.

## What Is an "Operation"?

This is the part that confused me at first. One operation is **one full execution of the code inside the benchmark loop**.

In the basic form (without `RunParallel`), the loop looks like:

```go
for i := 0; i < b.N; i++ {
    m.Set(i%10000, i)
}
```

Everything inside the `for b.N` loop is one operation: the `Set` call, the `i%10000` modulo, the `i++`, all of it.

With `b.RunParallel`, it's the same concept but the iterations are split across goroutines:

```go
b.RunParallel(func(pb *testing.PB) {
    i := 0
    for pb.Next() {
        // --- this is one operation ---
        m.Set(i%10000, i)
        // -----------------------------
        i++
    }
})
```

`pb.Next()` is the parallel equivalent of `i < b.N`. Each goroutine claims a share of the total iterations.

The framework measures total wall-clock time, divides by total iterations across all goroutines, and gives you `ns/op`.
It doesn't separate "your code" from "bookkeeping." The `i++` and modulo are included.
But they're so cheap (~0.3 ns each) compared to the map operation (~200 ns) that they're negligible noise.

## ns/op

```
BenchmarkSet_1Shard-10     5849028    208.0 ns/op
BenchmarkSet_64Shards-10   20112658   61.54 ns/op
```

**`ns/op`** is nanoseconds per operation. This is the main metric. Lower is better.

208 ns vs 61.5 ns: the 64-shard version is **3.4x faster** for `Set()`.

For `Get()` the gap is even wider:

```
BenchmarkGet_1Shard-10     9219829    145.6 ns/op
BenchmarkGet_64Shards-10   56095298   21.06 ns/op
```

145.6 ns vs 21 ns: **6.8x faster** with 64 shards.
Get improves more than Set because `RLock()` allows concurrent readers. With 64 shards and 10 goroutines, readers almost never block each other.

## B/op and allocs/op

These appear because we passed `-benchmem`:

```
BenchmarkSet_1Shard-10    ... 0 B/op    0 allocs/op
```

- **`B/op`**: bytes allocated on the **heap** per operation
- **`allocs/op`**: number of distinct **heap** allocations per operation

Both are `0` here. That's exactly what the kata asks for: "Zero Allocation Hot-Path."

If you ever see something like:

```
48 B/op    2 allocs/op
```

That means each operation allocated 48 bytes across 2 heap allocations.
At high throughput (the kata scenario is 50k+ RPS), those allocations create garbage collector pressure and eventually slow you down.

Common sources of allocations in Go:
- `[]byte(myString)`: converting a string to bytes allocates a copy
- `fmt.Sprintf(...)`: allocates the resulting string
- `interface{}` boxing: wrapping a concrete type in an interface can allocate
- `fnv.New64a()`: creates a new hasher on the heap each call

The last two are relevant to this kata. If you used `hash/fnv` with `New64a()` in the hot path, you'd see allocations show up here.

## Custom Metrics

Notice this field in the Set benchmarks:

```
10.00 goroutines
```

This isn't a built-in metric. The benchmark code explicitly reports it:

```go
b.ReportMetric(float64(numGoroutines), "goroutines")
```

You can add any custom metric with `b.ReportMetric`. Useful for tracking things like "items processed per second" or "cache hit ratio" alongside the standard timing.

## Why -count=3 Matters

Each benchmark appears 3 times in the output:

```
BenchmarkSet_1Shard-10     5849028    208.0 ns/op
BenchmarkSet_1Shard-10     5807408    207.8 ns/op
BenchmarkSet_1Shard-10     5823493    208.7 ns/op
```

These three runs are consistent: 208.0, 207.8, 208.7 ns/op. The variation is less than 1%.
This tells you the result is stable.

If the numbers jumped around (say, 200, 350, 180), something is wrong. Maybe CPU throttling, background processes, or thermal issues.
Consistent numbers mean you can trust the result.

For publishing or comparing benchmarks, use `-count=6` or higher. More samples give `benchstat` (covered below) enough data to calculate confidence intervals.

## Using benchstat to Compare Runs

Raw benchmark output is useful. But the real power comes from comparison. That's what `benchstat` does.

Install it:

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

### Reading a Single File

Save benchmark output to a file:

```bash
go test -bench=. -benchmem -count=6 > results.txt
```

Then run:

```bash
benchstat results.txt
```

Output:

```
│ results.txt │         │   sec/op    │

Set_1Shard-10             208.0n ± ∞ ¹
Set_64Shards-10           61.56n ± ∞ ¹
Get_1Shard-10             143.7n ± ∞ ¹
Get_64Shards-10           21.06n ± ∞ ¹
geomean                   66.37n
¹ need >= 6 samples for confidence interval at level 0.95
```

A few things to notice:

**`± ∞`** means the confidence interval couldn't be calculated.
With only 3 samples (our `-count=3`), there isn't enough data. The footnote says you need at least 6 samples for a 95% confidence interval.

**`geomean: 66.37n`** is the geometric mean across all benchmarks.
This gives you a single "overall performance" number. It's most useful when comparing two runs.

### Comparing Two Runs

The main use case for `benchstat` is comparing before and after a change:

```bash
# Before your change
go test -bench=. -benchmem -count=6 > old.txt

# Make changes to the code

# After your change
go test -bench=. -benchmem -count=6 > new.txt

# Compare
benchstat old.txt new.txt
```

This gives output like:

```       
│ old.txt    │   new.txt  │ │   sec/op    │ sec/op vs base │

Set_1Shard-10   208.0n ± 1%   195.0n ± 2%  -6.25% (p=0.002)
```

Three important fields:
- **vs base**: the percentage change (negative means faster)
- **p=0.002**: the p-value. Below 0.05 means the difference is statistically significant, not just noise
- **± 1%**: the confidence interval, showing how stable the measurements are

If `benchstat` reports `~ (p=0.342)` instead of a percentage, it means the difference is not statistically significant. Don't read into noise.

## Putting It All Together

Let's read one final line with everything we've learned:

```
BenchmarkSet_64Shards-10   20112658   61.54 ns/op   10.00 goroutines   0 B/op   0 allocs/op
```

This says: using a sharded map with 64 shards, running 10 goroutines in parallel, each `Set()` call took 61.5 nanoseconds on average, across 20 million iterations, with zero heap allocations.

Compared to 1 shard (208 ns/op), that's 3.4x faster. The lock contention dropped because 10 goroutines are no longer fighting over a single mutex; they're spread across 64 independent shards.

For `Get()`, the improvement is even larger (6.8x) because `RLock` lets multiple readers proceed without blocking each other.

These are the numbers the kata asks you to verify. The benchmark output tells the full story.

In [Part Two](/posts/reading-go-benchmark-output-cpu), we'll look at CPU profiling with `pprof` to see *where* the time is spent, and why 1 shard wastes 85% of CPU time on lock contention instead of doing useful work.
