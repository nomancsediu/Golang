# Garbage Collection (GC) in Go

## What Is Garbage Collection?

**Garbage collection** (GC) is automatic memory management. When you create objects on the heap, eventually you do not need them anymore. The garbage collector finds these unused objects and frees their memory so it can be reused.

In languages like C, you must manually free memory with `free()`. If you forget, you get a **memory leak**. If you free too early, you get a **use-after-free** bug. Both are serious problems.

Go handles this automatically. You allocate memory, and when it is no longer reachable, the GC cleans it up. No manual freeing needed.

```go
package main

import "fmt"

func createTempData() {
    // This slice is allocated on the heap
    data := make([]int, 1000)
    data[0] = 42
    fmt.Println("Data created:", data[0])
    // When this function returns, 'data' is no longer reachable
    // The GC will eventually free this memory
}

func main() {
    for i := 0; i < 10; i++ {
        createTempData()
    }
    fmt.Println("Done")
}
```

Each call to `createTempData()` creates a slice. After the function returns, the slice is no longer reachable. The GC will clean it up.

## Go's GC: Concurrent Tri-Color Mark-and-Sweep

Go uses a **concurrent tri-color mark-and-sweep** garbage collector. Let me break that down:

### Tri-Color

The GC uses three colors to categorize objects:

- **White**: objects that have not been visited yet (candidates for collection)
- **Gray**: objects that have been visited but their references have not been scanned yet
- **Black**: objects that have been visited and all their references have been scanned (definitely reachable)

### Mark Phase

The mark phase finds all **reachable** objects:

1. Start with root objects (global variables, stack variables, etc.) and color them **gray**
2. Pick a gray object, scan its references, and color those references gray
3. Color the original object **black**
4. Repeat until there are no more gray objects
5. All remaining **white** objects are unreachable

### Sweep Phase

The sweep phase frees all **white** (unreachable) objects:

1. Go through all objects in the heap
2. Any object still white is unreachable and can be freed
3. The memory is returned to the pool for future allocations

### Concurrent

The key word is **concurrent**. Go's GC runs alongside your program. It does not stop the world for long periods. This means your program keeps running while the GC does its work.

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // Start a goroutine that creates garbage
    go func() {
        for {
            _ = make([]byte, 1024) // create garbage
            time.Sleep(time.Millisecond)
        }
    }()

    // Your program keeps running smoothly
    // while the GC cleans up in the background
    for i := 0; i < 20; i++ {
        fmt.Println("Working...", i)
        time.Sleep(100 * time.Millisecond)
    }

    fmt.Println("Goroutines:", runtime.NumGoroutine())
}
```

## GC Pauses Are Very Short

Go's GC is designed for **low latency**. The stop-the-world pauses are typically under **1 millisecond**. This is impressive compared to older garbage collectors that could pause for hundreds of milliseconds.

Go achieves this by doing most of the GC work concurrently with your program. The stop-the-world phase only handles a small amount of synchronization work.

## Forcing a Garbage Collection

You can force a garbage collection using `runtime.GC()`. This is mostly useful for testing and benchmarking, not for production code.

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    var memBefore runtime.MemStats
    runtime.ReadMemStats(&memBefore)
    fmt.Printf("Before: Alloc = %d KB\n", memBefore.Alloc/1024)

    // Create some garbage
    for i := 0; i < 100000; i++ {
        _ = make([]byte, 100)
    }

    var memAfterAlloc runtime.MemStats
    runtime.ReadMemStats(&memAfterAlloc)
    fmt.Printf("After allocation: Alloc = %d KB\n", memAfterAlloc.Alloc/1024)

    // Force garbage collection
    runtime.GC()

    var memAfterGC runtime.MemStats
    runtime.ReadMemStats(&memAfterGC)
    fmt.Printf("After GC: Alloc = %d KB\n", memAfterGC.Alloc/1024)
}
```

This shows how much memory is allocated before and after GC.

## Monitoring GC with runtime.MemStats

The `runtime.MemStats` struct gives you detailed information about memory usage and GC behavior:

```go
package main

import (
    "fmt"
    "runtime"
)

func printMemStats(label string) {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("[%s]\n", label)
    fmt.Printf("  Alloc:      %d KB\n", m.Alloc/1024)
    fmt.Printf("  TotalAlloc: %d KB\n", m.TotalAlloc/1024)
    fmt.Printf("  Sys:        %d KB\n", m.Sys/1024)
    fmt.Printf("  NumGC:      %d\n", m.NumGC)
    fmt.Printf("  PauseTotalNs: %d ns\n", m.PauseTotalNs)
}

func main() {
    printMemStats("Start")

    // Allocate some data
    data := make([][]byte, 1000)
    for i := range data {
        data[i] = make([]byte, 1024)
    }

    printMemStats("After allocation")

    // Clear references
    data = nil

    // Force GC
    runtime.GC()

    printMemStats("After GC")
}
```

Key fields in `MemStats`:

- **Alloc**: currently allocated heap memory
- **TotalAlloc**: total memory allocated over the program's lifetime (cumulative)
- **Sys**: total memory obtained from the OS
- **NumGC**: number of completed GC cycles
- **PauseTotalNs**: total time spent in GC pauses

## GC Tuning with GOGC

Go's GC is controlled by the **GOGC** environment variable. It determines how frequently the GC runs.

The default value is **GOGC=100**, which means the GC runs when the heap grows by 100% since the last collection. In other words, if the heap was 10MB after the last GC, the next GC will run when it reaches 20MB.

```bash
# Default: GC runs when heap doubles
GOGC=100 go run main.go

# More frequent GC (less memory, more CPU)
GOGC=50 go run main.go

# Less frequent GC (more memory, less CPU)
GOGC=200 go run main.go

# Disable GC entirely (not recommended for production)
GOGC=off go run main.go
```

You can also set GOGC programmatically:

```go
package main

import (
    "fmt"
    "runtime/debug"
)

func main() {
    // Set GOGC to 50 (more aggressive GC)
    debug.SetGCPercent(50)
    fmt.Println("GOGC set to 50")

    // Your program runs here
}
```

## Understanding Memory Leaks in Go

Even with garbage collection, you can have **memory leaks** in Go. A memory leak happens when you hold references to data you no longer need. The GC cannot free memory that is still reachable.

```go
package main

import "fmt"

// This is a leak! We keep adding to the cache but never remove entries
var cache = map[string][]byte{}

func addToCache(key string, data []byte) {
    cache[key] = data // data is now reachable through cache
    // The GC cannot free it because cache still references it
}

func main() {
    for i := 0; i < 10000; i++ {
        key := fmt.Sprintf("key-%d", i)
        data := make([]byte, 1024)
        addToCache(key, data)
    }

    fmt.Println("Cache size:", len(cache))
    // All 10000 entries are still in memory!
}
```

The fix is to remove entries you no longer need:

```go
package main

import "fmt"

var cache = map[string][]byte{}

func addToCache(key string, data []byte) {
    // Remove old entries when cache gets too big
    if len(cache) > 1000 {
        // Simple eviction: clear the whole cache
        cache = make(map[string][]byte)
    }
    cache[key] = data
}

func main() {
    for i := 0; i < 10000; i++ {
        key := fmt.Sprintf("key-%d", i)
        data := make([]byte, 1024)
        addToCache(key, data)
    }

    fmt.Println("Cache size:", len(cache))
}
```

## Why Understanding GC Matters

1. **Performance tuning**: knowing how GC works helps you tune GOGC and reduce pause times
2. **Memory leaks**: understanding that GC only frees unreachable objects helps you find leaks
3. **Allocation patterns**: knowing that heap allocations trigger GC helps you minimize them
4. **Debugging**: when your program has memory issues, understanding GC helps you diagnose them
5. **Production readiness**: for high-performance services, GC behavior affects latency and throughput

## Key Takeaways

- **Garbage collection** automatically frees unreachable heap memory
- Go uses a **concurrent tri-color mark-and-sweep** algorithm
- The **mark phase** finds reachable objects; the **sweep phase** frees unreachable ones
- GC runs **concurrently** with your program, with very short pauses
- Use `runtime.GC()` to force a collection (mainly for testing)
- Use `runtime.MemStats` to monitor memory usage
- **GOGC** controls how frequently the GC runs (default: 100)
- Memory leaks in Go happen when you hold references to data you no longer need
- Understanding GC helps with performance tuning, debugging, and writing efficient code

I used to think garbage collection meant I never had to think about memory. That was wrong. Understanding GC made me realize that I still need to be mindful about what I keep in memory. The GC is not magic; it can only free what is unreachable. If you hold onto things, they stay.

---

*Next: [SP vs BP](ch02-12-sp-vs-bp.md)*
