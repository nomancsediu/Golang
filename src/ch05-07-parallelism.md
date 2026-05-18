# Parallelism

## Doing Multiple Things at the Same Time

If concurrency is about dealing with multiple things at once, **parallelism** is about actually doing multiple things at the exact same time. Parallelism requires hardware support: you need multiple CPU cores (or multiple processors) to execute multiple instructions simultaneously.

I used to think that making my program concurrent would automatically make it faster. It does not. Concurrency is about structure; parallelism is about execution speed. You need both for maximum performance.

## What Is Parallelism?

**Parallelism** is when multiple operations are executed at the exact same moment on different CPU cores. There is no time-sharing, no switching. They are truly simultaneous.

Going back to our cooking analogy: if concurrency is one chef handling multiple dishes by switching between them, parallelism is **multiple chefs** each working on their own dish at the same time.

```
Single chef (concurrent, not parallel):
Time:    0ms    200ms   400ms   600ms   800ms
Chef 1:  [Pasta] [Sauce] [Pasta] [Sauce] [Serve]
              ^       ^       ^
           switch   switch   switch

Two chefs (parallel):
Time:    0ms    200ms   400ms   600ms
Chef 1:  [Pasta  Pasta  Pasta  Serve]
Chef 2:  [Sauce  Sauce  Sauce  Done ]
         ^--- both working at same time ---^
```

With two chefs (two CPU cores), both tasks happen at the same time, so the total time is roughly half.

## Parallelism Requires Multiple Cores

This is obvious but worth stating: **you cannot have parallelism on a single CPU core**. A single core can only execute one instruction at a time. It can be concurrent (switching between tasks), but not parallel.

Modern computers have multiple cores. My laptop has 8 cores. A server might have 64 or more. Each core can execute its own instruction stream independently.

```
+---------------------------------------------------+
|                     CPU Chip                       |
|                                                    |
|  +----------+  +----------+  +----------+  +-----+|
|  |  Core 0  |  |  Core 1  |  |  Core 2  |  |Core3||
|  |          |  |          |  |          |  |     ||
|  | ALU      |  | ALU      |  | ALU      |  | ALU ||
|  | L1 Cache |  | L1 Cache |  | L1 Cache |  | L1  ||
|  | L2 Cache |  | L2 Cache |  | L2 Cache |  | L2  ||
|  +----------+  +----------+  +----------+  +-----+|
|                                                    |
|              +------------------+                  |
|              |    L3 Cache      |                  |
|              |   (shared)       |                  |
|              +------------------+                  |
+---------------------------------------------------+
```

Each core runs independently. They share L3 cache and main memory, which means they need to coordinate access to shared data (this is where synchronization becomes important).

## Concurrent + Multicore = Parallel Execution

Here is where it all comes together. If you write a concurrent program (with goroutines) and run it on a multicore machine, the Go runtime will schedule goroutines across multiple cores. This means your concurrent tasks actually run in parallel.

Go does this automatically. The Go runtime has a scheduler that distributes goroutines across OS threads, and the OS schedules those threads across CPU cores.

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func heavyWork(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    start := time.Now()
    // Simulate CPU-intensive work
    total := 0
    for i := 0; i < 100_000_000; i++ {
        total += i
    }
    fmt.Printf("Worker %d finished in %v (result: %d)\n", id, time.Since(start), total)
}

func main() {
    numWorkers := 4
    var wg sync.WaitGroup

    fmt.Printf("CPU cores available: %d\n", runtime.NumCPU())
    fmt.Printf("Running with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println()

    // Run with multiple cores (parallel)
    fmt.Println("=== Parallel (using all cores) ===")
    start := time.Now()
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go heavyWork(i, &wg)
    }
    wg.Wait()
    parallelTime := time.Since(start)
    fmt.Printf("Total time: %v\n\n", parallelTime)

    // Run with single core (concurrent but not parallel)
    fmt.Println("=== Single core (concurrent only) ===")
    runtime.GOMAXPROCS(1)
    start = time.Now()
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go heavyWork(i, &wg)
    }
    wg.Wait()
    singleCoreTime := time.Since(start)
    fmt.Printf("Total time: %v\n\n", singleCoreTime)

    fmt.Printf("Speedup from parallelism: %.2fx\n",
        float64(singleCoreTime)/float64(parallelTime))
}
```

When I run this on my 8-core laptop, the parallel version is about 3 to 4 times faster than the single-core version. Not 4 times faster (which would be perfect scaling), because there is overhead from scheduling, memory access, and other factors. But it is significantly faster.

## Amdahl's Law

There is a fundamental limit to how much parallelism can speed up your program. It is called **Amdahl's Law**, and it says that the speedup is limited by the sequential portion of your program.

If 90% of your program can be parallelized and 10% must be sequential, the maximum speedup you can get (even with infinite cores) is:

```
Speedup = 1 / (sequential_fraction + parallel_fraction / num_cores)
        = 1 / (0.1 + 0.9 / infinity)
        = 1 / 0.1
        = 10x
```

So even with infinite cores, you can only get a 10x speedup. The 10% sequential part is the bottleneck.

```
+--------------------------------------------------+
|  Amdahl's Law: Maximum Speedup                   |
|                                                   |
|  90% parallelizable:  max speedup = 10x          |
|  75% parallelizable:  max speedup = 4x           |
|  50% parallelizable:  max speedup = 2x           |
|  25% parallelizable:  max speedup = 1.33x        |
|                                                   |
|  More cores helps, but sequential code is the     |
|  real bottleneck.                                 |
+--------------------------------------------------+
```

**The lesson**: Before trying to parallelize everything, identify the sequential bottlenecks. Sometimes making the sequential code faster gives you more benefit than adding more parallelism.

## When Parallelism Helps (and When It Does Not)

**Parallelism helps with CPU-bound tasks:**
- Number crunching and math operations
- Image and video processing
- Data compression and encryption
- Sorting and searching large datasets

**Parallelism does not help much with I/O-bound tasks:**
- Reading files from disk
- Making network requests
- Waiting for database queries
- User input handling

For I/O-bound tasks, concurrency (not parallelism) is what you need. A single core can handle thousands of concurrent I/O operations by switching between them while they wait.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func ioBoundTask(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    // Simulate I/O wait (network, disk, database)
    time.Sleep(100 * time.Millisecond)
    fmt.Printf("I/O task %d done\n", id)
}

func cpuBoundTask(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    // Simulate CPU work
    total := 0
    for i := 0; i < 50_000_000; i++ {
        total += i
    }
    fmt.Printf("CPU task %d done (sum=%d)\n", id, total)
}

func main() {
    const numTasks = 8
    var wg sync.WaitGroup

    // I/O-bound tasks: parallelism barely helps
    fmt.Println("=== I/O-bound tasks (mostly waiting) ===")
    start := time.Now()
    for i := 0; i < numTasks; i++ {
        wg.Add(1)
        go ioBoundTask(i, &wg)
    }
    wg.Wait()
    fmt.Printf("I/O tasks total time: %v\n\n", time.Since(start))

    // CPU-bound tasks: parallelism helps a lot
    fmt.Println("=== CPU-bound tasks (actual computation) ===")
    start = time.Now()
    for i := 0; i < numTasks; i++ {
        wg.Add(1)
        go cpuBoundTask(i, &wg)
    }
    wg.Wait()
    fmt.Printf("CPU tasks total time: %v\n", time.Since(start))
}
```

The I/O tasks finish in about 100ms regardless of how many cores you have, because they are just waiting. The CPU tasks are much faster with more cores.

## Go Runtime Schedules Goroutines Across OS Threads

One of the best things about Go is that you do not have to manually manage parallelism. You just write concurrent code using goroutines, and the Go runtime automatically distributes them across OS threads, which the OS then schedules across CPU cores.

The `GOMAXPROCS` setting controls how many OS threads the Go runtime uses for executing goroutines. By default (since Go 1.5), it is set to the number of CPU cores, which means Go will use all available cores for parallel execution.

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // Check current GOMAXPROCS value
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("CPU cores: %d\n", runtime.NumCPU())

    // You can change it (usually you should not)
    // runtime.GOMAXPROCS(1) // force single-core mode
    // runtime.GOMAXPROCS(4) // use 4 cores
}
```

In almost all cases, you should leave `GOMAXPROCS` at the default value. The Go runtime is very good at scheduling goroutines efficiently across available cores.

## The Takeaway

Parallelism is about speed through simultaneous execution. It requires multiple CPU cores and works best for CPU-bound tasks. Go makes it easy: write concurrent code with goroutines, and the runtime automatically parallelizes it across available cores. But remember Amdahl's Law: the sequential parts of your program will always limit the maximum speedup. Focus on what matters most.
