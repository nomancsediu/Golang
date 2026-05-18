# Runtime Internals

## What is the Go Runtime?

The **Go runtime** is the support system that runs alongside your Go code. It handles memory allocation, garbage collection, goroutine scheduling, and more. Understanding the runtime helps you write better, faster, and more reliable Go programs.

The runtime is not something you interact with directly most of the time. But knowing how it works helps you debug problems, optimize performance, and make better design decisions.

## Go Runtime Components

### Scheduler

The Go scheduler manages goroutines. It decides which goroutine runs on which OS thread. It uses the **GMP model**:

- **G** (Goroutine) - A goroutine waiting to run
- **M** (Machine) - An OS thread
- **P** (Processor) - A logical processor that holds a run queue of goroutines

When a goroutine makes a blocking system call, the scheduler detaches the P from the M and gives it to another M. This way, other goroutines can keep running while one is blocked.

### Garbage Collector

Go uses a **concurrent, tri-color mark-and-sweep** garbage collector. It runs alongside your program without stopping the world (mostly).

The GC cycle:
1. **Mark phase** - Traverse all reachable objects starting from roots (stacks, globals). Mark them as alive.
2. **Sweep phase** - Free all unmarked objects.

The three colors:
- **White** - Not yet visited (candidates for collection)
- **Grey** - Visited but children not yet scanned
- **Black** - Visited and all children scanned (definitely alive)

You can tune the GC with the `GOGC` environment variable. The default is 100, which means the GC runs when the heap grows by 100% since the last collection. Setting `GOGC=200` means the GC waits until the heap triples.

### Memory Allocator

Go's memory allocator uses **TCMalloc** (Thread-Caching Malloc) as its inspiration. It divides memory into different size classes and uses per-thread caches to reduce lock contention:

- Small objects (under 32 KB) are allocated from per-P caches
- Large objects are allocated directly from the heap
- This reduces fragmentation and improves allocation speed

### Goroutine Manager

The runtime manages goroutine lifecycle:
- Creating goroutines (stack allocation, scheduling)
- Growing and shrinking stacks
- Parking and unparking goroutines
- Tracking goroutine state for debuggers

## How Go Starts

When you run a Go program, the following happens:

1. **runtime.init()** - Initialize the runtime (scheduler, memory allocator, GC)
2. **Package init()** - Run all init() functions in dependency order
3. **main.main()** - Your program starts

```go
// This runs first
func init() {
    // Package initialization
}

// This runs after all init() functions
func main() {
    // Your code
}
```

Each package can have multiple init() functions. They run in the order they appear in the file, and packages are initialized based on their import order.

## Stack Management

Go goroutines start with a small stack (2 KB on most systems). The stack grows and shrinks dynamically:

```go
func deepRecursion(n int) int {
    if n <= 0 {
        return 0
    }
    // Each recursive call might trigger a stack growth
    return 1 + deepRecursion(n-1)
}
```

**Stack growth**: When a goroutine needs more stack space, the runtime allocates a new, larger stack and copies the existing data. This is called **stack splitting**. It happens transparently.

**Stack shrinking**: When a goroutine uses less stack than allocated, the runtime shrinks it. This happens during garbage collection.

This is why Go can create millions of goroutines. They start tiny and only grow as needed.

## Garbage Collection Triggers

The GC can be triggered by:

1. **Heap size** - When the heap grows past the GOGC threshold
2. **Time** - After 2 minutes of no GC (forced collection)
3. **Manual** - Calling `runtime.GC()`

```go
// Force a garbage collection (rarely needed)
runtime.GC()

// Read GC statistics
var stats debug.GCStats
debug.ReadGCStats(&stats)
fmt.Printf("Last GC: %v\n", stats.LastGC)
fmt.Printf("GC pauses: %v\n", stats.PauseTotal)
```

You can also use `runtime/debug` to adjust GC parameters at runtime:

```go
// Set GC target percentage
debug.SetGCPercent(200)

// Set memory limit (Go 1.19+)
debug.SetMemoryLimit(1 << 30) // 1 GB
```

## Channel Internals

A Go channel is implemented as a **hselect structure** containing:

- **Circular queue** (buffer) - For buffered channels
- **Mutex** - Protects the channel state
- **Send queue** (sudog list) - Goroutines waiting to send
- **Receive queue** (sudog list) - Goroutines waiting to receive

When you send to a full channel:
1. The goroutine is wrapped in a **sudog** structure
2. It is added to the send queue
3. The goroutine is parked (put to sleep)
4. When a receiver takes a value, the sender is unparked

When you receive from an empty channel:
1. The goroutine is wrapped in a sudog
2. It is added to the receive queue
3. The goroutine is parked
4. When a sender provides a value, the receiver is unparked

This is why channels are safe for concurrent use without additional locking.

## Runtime Inspection

You can inspect the runtime from within your Go program:

```go
func inspectRuntime() {
    // Number of goroutines
    fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())

    // Number of CPU cores
    fmt.Printf("CPUs: %d\n", runtime.NumCPU())

    // Memory statistics
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Alloc: %d KB\n", m.Alloc/1024)
    fmt.Printf("TotalAlloc: %d KB\n", m.TotalAlloc/1024)
    fmt.Printf("Sys: %d KB\n", m.Sys/1024)
    fmt.Printf("NumGC: %d\n", m.NumGC)
    fmt.Printf("Goroutines: %d\n", m.NumGoroutine)

    // GC percentage
    fmt.Printf("GOGC: %d\n", debug.SetGCPercent(-1))
    debug.SetGCPercent(100) // restore default
}
```

## Stack Trace

When a goroutine panics or you send a SIGQUIT, Go prints a stack trace for every goroutine. This is invaluable for debugging:

```go
// Print stack traces of all goroutines
buf := make([]byte, 1<<20)
n := runtime.Stack(buf, true) // true = all goroutines
fmt.Printf("%s\n", buf[:n])
```

## Why Understanding Runtime Helps

**Debugging** - When your program is slow, knowing about the GC helps you identify if garbage collection is the bottleneck. When goroutines are stuck, knowing about the scheduler helps you find the deadlock.

**Performance** - Understanding stack growth helps you avoid unnecessary allocations. Understanding the allocator helps you reduce memory pressure.

**Reliability** - Knowing that goroutines can leak helps you design cleanup mechanisms. Knowing that channels block helps you avoid deadlocks.

You do not need to memorize every detail of the runtime. But having a mental model of how it works makes you a much more effective Go developer. When something goes wrong, you know where to look.
