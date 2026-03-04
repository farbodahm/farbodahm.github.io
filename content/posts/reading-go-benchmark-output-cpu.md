---
author: "Farbod Ahmadian"
title: "Reading Go Benchmark Output (Part Two): CPU Profiling"
date: "2026-03-04"
description: "How to generate and read CPU profiles in Go using pprof, with real examples showing lock contention in a concurrent sharded map."
tags:
- golang
- benchmarking
- performance
---

In [Part One](/posts/reading-go-benchmark-output-memory), we learned how to read Go benchmark output: what `ns/op`, `B/op`, iterations, and `GOMAXPROCS` mean.
We saw that a sharded map with 64 shards is 3.4x faster than 1 shard for writes.

But *why* is it faster? The benchmark output tells you *how fast* something is. CPU profiling tells you *where the time goes*.

All examples in this post come from the same [Concurrent Map with Sharded Locks Go Kata](https://github.com/farbodahm/go-kata/pull/3/) as Part One.

## Generating a CPU Profile

To generate a CPU profile, add the `-cpuprofile` flag to your benchmark command:

```bash
go test -bench=BenchmarkSet_1Shard -cpuprofile=cpu_1shard.prof
```

This runs the benchmark and writes a binary profile to `cpu_1shard.prof`.
The file is not human-readable. You need `go tool pprof` to analyze it.

Note: target a specific benchmark with `-bench=BenchmarkSet_1Shard` instead of `-bench=.`.
If you profile all benchmarks at once, the profile mixes samples from different benchmarks and becomes harder to interpret.

For our comparison, we generate two profiles:

```bash
go test -bench=BenchmarkSet_1Shard -cpuprofile=cpu_1shard.prof
go test -bench=BenchmarkSet_64Shards -cpuprofile=cpu_64shards.prof
```

## Reading the Profile with pprof -top

The first thing to do with a profile is look at the top CPU consumers:

```bash
go tool pprof -top cpu_1shard.prof
```

Here's the output for 1 shard (trimmed for readability):

```
Type: cpu
Duration: 1.80s, Total samples = 5.46s (302.88%)
Showing nodes accounting for 5.30s, 97.07% of 5.46s total

      flat  flat%   sum%        cum   cum%
     2.18s 39.93% 39.93%      2.18s 39.93%  runtime.usleep
     1.64s 30.04% 69.96%      1.64s 30.04%  runtime.pthread_cond_wait
     0.84s 15.38% 85.35%      0.84s 15.38%  runtime.pthread_cond_signal
     0.26s  4.76% 90.11%      1.10s 20.15%  internal/sync.(*Mutex).lockSlow
     0.08s  1.47% 91.58%      0.08s  1.47%  sync/atomic.(*Int32).Add
     0.03s  0.55% 95.97%      0.07s  1.28%  runtime.mapassign_fast64
```

That's a lot of numbers. Let's break down each part.

## The Header: Duration and Total Samples

```
Duration: 1.80s, Total samples = 5.46s (302.88%)
```

Three values:

- **Duration: 1.80s** — wall-clock time the benchmark ran
- **Total samples: 5.46s** — sum of CPU time across *all cores*
- **302.88%** — Total samples / Duration. This tells you CPU utilization

The percentage is key. **302%** means the benchmark used about 3 out of 10 available cores on average. The other 7 cores were mostly idle.

That's already a red flag for a parallel benchmark running with `GOMAXPROCS=10`. We have 10 goroutines but only 3 cores worth of work is happening. Where are the other 7? Blocked on locks.

Compare this to the 64-shard profile header:

```
Duration: 1.60s, Total samples = 8.49s (529.43%)
```

**529%** means about 5.3 cores utilized. Almost double the 1-shard case.
The same benchmark, same code structure, same `GOMAXPROCS`. The only difference is shard count. More shards means less contention, which means goroutines spend more time doing real work instead of waiting.

## flat vs cum

Every row in the `pprof -top` output has two important metrics: **flat** and **cum** (cumulative).

```
      flat  flat%   sum%        cum   cum%
     0.26s  4.76% 90.11%      1.10s 20.15%  internal/sync.(*Mutex).lockSlow
```

- **flat (0.26s)** — time spent *directly executing code in that function*, not counting anything it calls
- **cum (1.10s)** — time spent *in that function plus everything it calls*

Think of it this way: if `lockSlow` calls `runtime.usleep` internally, the time spent sleeping counts toward `lockSlow`'s **cum** but not its **flat**. The flat time of `usleep` only shows up on `usleep`'s own row.

There are also percentage columns:
- **flat%** — flat time as percentage of total samples
- **cum%** — cumulative time as percentage of total samples
- **sum%** — running total of flat% as you go down the list. Useful to see how much the top N functions account for

## Reading the 1-Shard Profile: A Contention Story

Now let's actually read the 1-shard profile. The top 3 functions by flat time:

```
      flat  flat%   sum%        cum   cum%
     2.18s 39.93% 39.93%      2.18s 39.93%  runtime.usleep
     1.64s 30.04% 69.96%      1.64s 30.04%  runtime.pthread_cond_wait
     0.84s 15.38% 85.35%      0.84s 15.38%  runtime.pthread_cond_signal
```

What are these functions?

- **`runtime.usleep`** — goroutine sleeping, waiting to be woken up
- **`runtime.pthread_cond_wait`** — OS-level blocking. A goroutine is parked, waiting for a mutex to become available
- **`runtime.pthread_cond_signal`** — waking up a blocked goroutine after a mutex is released

**85% of CPU time is spent on lock synchronization.** Goroutines are sleeping, waiting, and signaling each other. That's not useful work.

Now look at the function that does the actual map write:

```
     0.03s  0.55%            0.07s  1.28%  runtime.mapassign_fast64
```

0.55% of CPU time. The code that *actually stores data in the map* gets barely any CPU because the goroutines spend most of their time fighting over the single lock.

If you sort by cumulative instead (`pprof -top -cum`), the picture is even clearer:

```
      flat  flat%   sum%        cum   cum%
         0     0%     0%      3.70s 67.77%  (*ShardedMap[...]).Set
         0     0%     0%      2.50s 45.79%  sync.(*RWMutex).Unlock
     0.03s  0.55%  0.92%      1.13s 20.70%  sync.(*Mutex).Lock
```

`Set()` consumes 67.77% of cumulative time. But `Set` itself has 0 flat time; it doesn't do anything directly. It just calls `Lock`, the map assignment, and `Unlock`. And of those:

- `RWMutex.Unlock` takes **45.79%** cumulative — almost half of all CPU time just releasing the lock
- `Mutex.Lock` takes **20.70%** cumulative — acquiring the lock

Together, lock acquire and release account for **66% of CPU time**. The actual work (assigning a value in the map) is a rounding error.

## Reading the 64-Shard Profile: Less Contention

Same analysis for 64 shards. The top flat consumers:

```
      flat  flat%   sum%        cum   cum%
     5.90s 69.49% 69.49%      5.90s 69.49%  runtime.usleep
     0.42s  4.95% 74.44%      0.42s  4.95%  sync/atomic.(*Int32).Add
     0.40s  4.71% 79.15%      0.40s  4.71%  runtime.pthread_cond_wait
     0.16s  1.88% 90.69%      0.84s  9.89%  sync.(*RWMutex).Lock
     0.12s  1.41% 92.11%      0.25s  2.94%  runtime.mapassign_fast64
```

The `usleep` number looks higher (5.90s vs 2.18s). But remember: **Total samples is also higher** (8.49s vs 5.46s).
This `usleep` is mostly the Go scheduler's idle threads. With `GOMAXPROCS=10` and work spread across 64 shards, threads finish their work fast and sleep between scheduling. It's not contention; it's efficiency.

The real indicator of contention is `lockSlow`, the slow path where a goroutine *actually has to wait* for a lock:

| | flat | cum |
|---|---|---|
| **1 shard** | 0.26s | **1.10s (20.15%)** |
| **64 shards** | 0.08s | **0.28s (3.30%)** |

Lock contention dropped from **20% to 3%** of CPU time.

And look at `mapassign_fast64`, the actual map write operation:

| | flat |
|---|---|
| **1 shard** | 0.03s (0.55%) |
| **64 shards** | 0.12s (1.41%) |

With 64 shards, the map write gets **4x more CPU time**. Not because it's slower; because goroutines actually reach it instead of being stuck waiting for locks.

## Zooming into a Function with pprof -list

`pprof -top` shows you which functions are hot. To see *which lines* inside a function are hot, use `-list`:

```bash
go tool pprof -list='ShardedMap.*Set' cpu_1shard.prof
```

This annotates the source code with CPU time per line:

```
Total: 5.46s
ROUTINE ======================== (*ShardedMap[...]).Set
         0      3.70s (flat, cum) 67.77% of Total
         .          .     45:func (m *ShardedMap[K, V]) Set(key K, value V) {
         .       10ms     46:   index := m.shardIndex(key)
         .      1.13s     47:   m.locks[index].Lock()
         .          .     48:   defer m.locks[index].Unlock()
         .          .     49:
         .       60ms     50:   m.shards[index][key] = value
         .      2.50s     51:}
```

Reading this line by line:

- **Line 5** (`shardIndex`): 10ms — computing the hash is almost free
- **Line 6** (`Lock`): 1.13s — acquiring the lock takes **30% of the function's time**
- **Line 9** (map write): 60ms — the actual work
- **Line 10** (closing brace): 2.50s — this is where `defer Unlock()` runs, **67% of the function's time**

The `defer m.locks[index].Unlock()` on line 48 doesn't show time; the deferred call executes at the function return on line 51. That's why line 51 has 2.50s.

Unlocking is more expensive than locking here because every unlock has to wake up a waiting goroutine. With 10 goroutines competing for 1 lock, there's always someone waiting.

Now the same view for 64 shards:

```
Total: 8.49s
ROUTINE ======================== (*ShardedMap[...]).Set
      60ms      4.62s (flat, cum) 54.42% of Total
      10ms       10ms     45:func (m *ShardedMap[K, V]) Set(key K, value V) {
      10ms       40ms     46:   index := m.shardIndex(key)
         .      840ms     47:   m.locks[index].Lock()
         .          .     48:   defer m.locks[index].Unlock()
         .          .     49:
         .      220ms     50:   m.shards[index][key] = value
      40ms      3.51s     51:}
```

Lock acquisition dropped from 1.13s to **840ms**. Map write went from 60ms to **220ms** (more actual work getting done). The function's share of total time dropped from 67.77% to 54.42%.

The numbers tell the same story from a different angle: less contention means more time doing real work.

## Visualizing the Profile with pprof -svg

Text output is precise but hard to scan. A visual call graph shows you the full picture at a glance.

Generate one with:

```bash
go tool pprof -svg -nodefraction=0.01 cpu_1shard.prof > profile_1shard.svg
```

The `-nodefraction=0.01` flag drops functions that account for less than 1% of total CPU time. Without it, pprof includes every tiny function and the graph becomes unreadable. Adjust the threshold depending on how much detail you want.

### The 1-Shard Graph

Here's the call graph for the 1-shard benchmark:

{{< scrollable-svg src="/images/pprof-1shard.svg" alt="pprof call graph for 1-shard benchmark showing most time spent in lock contention" height="500px" >}}

Let's break down how to read it.

**Boxes** are functions. Each box shows the function name, flat time, and cumulative time. Bigger and more red a box is, the more CPU time it consumed. The three largest boxes at the bottom are `runtime.usleep` (2.18s, 39.93%), `runtime.pthread_cond_wait` (1.64s, 30.04%), and `runtime.pthread_cond_signal` (0.84s, 15.38%). These are the lock contention functions we saw in `-top`.

**Arrows** are function calls. The label on each arrow shows how much time flows through that call. Thicker arrows mean more time. Follow the thick arrows from top to bottom and you trace the hot path.

**The hot path**: Start at `testing.(*B).RunParallel.func1` at the top (3.58s). It calls `benchmarkShardedSet.func1` (3.58s), which calls `Set` (3.70s). From `Set`, the thick arrows fan out to `sync.(*RWMutex).Unlock` (2.50s) and `sync.(*RWMutex).Lock` (1.13s). A thin arrow goes to `runtime.mapassign_fast64` (0.07s) — the actual map write. The arrow thickness tells the story visually: the lock operations dwarf the useful work.

From Lock and Unlock, the arrows flow down through `internal/sync.(*Mutex).lockSlow` (1.10s cum) and eventually into the big red boxes: `usleep`, `pthread_cond_wait`, and `pthread_cond_signal`. This is where goroutines are parked waiting for the single mutex.

### The 64-Shard Graph

Now compare with the 64-shard graph:

{{< scrollable-svg src="/images/pprof-64shards.svg" alt="pprof call graph for 64-shard benchmark showing reduced lock contention" height="500px" >}}

The structure is different. The dominant box is `runtime.usleep` (5.90s, 69.49%) — but this is mostly idle scheduler threads, not contention. You can tell because:

1. The arrows leading to `usleep` come from `runtime.schedule` and `runtime.findRunnable` (the Go scheduler looking for work), not from mutex operations
2. `lockSlow` is barely visible in the graph — it's a tiny box compared to the 1-shard version
3. `mapassign_fast64` (0.12s) appears as a visible box now, showing that real work is actually happening

The `Set` function (4.62s cum) still takes significant cumulative time, but more of it flows into actual map operations (0.22s) rather than lock waiting. The `sync.(*Mutex).Lock` box (0.36s flat, 0.64s cum) is much smaller than in the 1-shard graph (where it was 0.01s flat but 1.14s cum via `RWMutex.Lock`).

### Reading Tips for pprof Graphs

A few patterns to look for:

- **Large red boxes at the bottom** usually indicate leaf functions where time is actually spent (sleeping, spinning, computing)
- **Many thick arrows converging** on a single node signals a bottleneck
- **Thin arrows to small boxes** are functions that barely register — they're not your problem
- **Wide fan-out from a node** (one function calling many others) suggests the function is an orchestrator; look at its children to find the real cost

## Other pprof Modes

### Interactive Mode

Run pprof without flags to get an interactive shell:

```bash
go tool pprof cpu_1shard.prof
```

This opens a `(pprof)` prompt where you can run commands like `top`, `list`, `web`, and more. Useful for exploration when you don't know what you're looking for yet.

### Web UI

If you have a browser available:

```bash
go tool pprof -http=:8080 cpu_1shard.prof
```

This opens an interactive web UI where you can switch between graph, flame graph, top, source, and disassembly views.
The flame graph view is particularly good for understanding nested call stacks.

## Summary

Memory benchmarking (`-benchmem`) tells you *how fast* and *how much allocation*. CPU profiling (`-cpuprofile`) tells you *why*.

For our sharded map, the CPU profile showed exactly what the [kata description](https://github.com/farbodahm/go-kata/pull/3/) predicted:

- **1 shard**: 85% of CPU time wasted on mutex waiting and signaling. Only 0.55% spent on actual map writes. 302% CPU utilization out of a possible 1000%
- **64 shards**: lock contention drops to 3%. Map writes get 4x more CPU time. CPU utilization rises to 529%

The commands to remember:

```bash
# Generate the profile
go test -bench=BenchmarkName -cpuprofile=cpu.prof

# See top CPU consumers
go tool pprof -top cpu.prof

# Sort by cumulative time (find the hot call chains)
go tool pprof -top -cum cpu.prof

# Zoom into a specific function's source lines
go tool pprof -list='FunctionName' cpu.prof

# Visual call graph
go tool pprof -svg cpu.prof > profile.svg

# Interactive web UI
go tool pprof -http=:8080 cpu.prof
```

Next time a benchmark looks slow, don't guess. Profile it. The numbers will tell you exactly where to look.
