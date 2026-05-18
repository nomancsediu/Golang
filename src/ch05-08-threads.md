# Threads

## The Building Blocks of Concurrent Programs

Before I learned about threads, I thought the only way to run things concurrently was to start a new process. But processes are heavy. They have their own memory space, their own file descriptors, their own everything. That is expensive. **Threads** are the lighter alternative.

## What Is a Thread?

A **thread** is the smallest unit of execution within a process. While a process is a container that holds code, data, and resources, a thread is what actually executes the code.

Every process has at least one thread (the main thread). When you run a Go program, the main function runs in the main thread of the process. You can create additional threads to run code concurrently within the same process.

The key difference between processes and threads is **sharing**:

```
+-------------------------------------------------+
|                    Process                       |
|                                                  |
|  +--------------------------------------------+ |
|  |         Shared between ALL threads          | |
|  |                                            | |
|  |  Code Segment (program instructions)       | |
|  |  Data Segment (global variables)           | |
|  |  Heap (dynamically allocated memory)       | |
|  |  File Descriptors (open files, sockets)    | |
|  |  Current working directory                 | |
|  +--------------------------------------------+ |
|                                                  |
|  +-----------------+  +-----------------+        |
|  |   Thread 1      |  |   Thread 2      |       |
|  |                 |  |                 |        |
|  |  Stack          |  |  Stack          |        |
|  |  Program Counter|  |  Program Counter|        |
|  |  Registers      |  |  Registers      |        |
|  +-----------------+  +-----------------+        |
|                                                  |
+-------------------------------------------------+
```

**Threads share:**
- Code segment (the program instructions)
- Data segment (global variables)
- Heap (all dynamically allocated memory)
- File descriptors (open files, network sockets)
- Process ID

**Each thread has its own:**
- Stack (local variables and function call frames)
- Program counter (which instruction it is executing)
- Registers (CPU register values)
- Thread ID

This sharing is both a blessing and a curse. It makes communication between threads fast (they can read the same variables), but it also creates the need for synchronization (what if two threads modify the same variable at the same time?).

## Threads vs Processes

Here is a comparison that really helped me understand the difference:

| Aspect           | Process                        | Thread                           |
|-----------------|-------------------------------|----------------------------------|
| Memory           | Separate address space         | Shared address space              |
| Creation cost    | Expensive (copy everything)    | Cheap (just stack + registers)   |
| Context switch   | Slow (swap memory mappings)    | Faster (same address space)      |
| Communication    | IPC (pipes, sockets, shared mem) | Direct memory access           |
| Safety           | Isolated (one crash does not affect others) | Shared (one bug can corrupt shared data) |
| Resource use     | High (each has own memory)     | Low (shared memory)              |

Creating a process involves duplicating the entire address space, setting up new page tables, and copying file descriptors. Creating a thread just needs a new stack and some register values. That is why thread creation is much faster.

## User-Level Threads vs Kernel-Level Threads

There are two types of threads, and the distinction is important:

**Kernel-level threads** are managed by the operating system. The OS knows about them, schedules them, and handles context switches. These are "real" threads as far as the OS is concerned.

**User-level threads** are managed by a runtime or library in user space. The OS does not know about them. The runtime decides when to switch between them. These are sometimes called "green threads" or "fibers."

```
Kernel-level threads:
+-------------------+        +-----------+
| User program      |        |           |
| Thread A <------->| syscall |   OS      |
| Thread B <------->|-------->|  Kernel   |
| Thread C <------->|        |  Scheduler|
+-------------------+        +-----------+
The OS sees and schedules each thread.

User-level threads:
+---------------------------+
| User program              |
|                           |
|  +---------------------+  |
|  | Runtime scheduler   |  |
|  | Green A <-> Green B |  |
|  | Green C <-> Green D |  |
|  +---------------------+  |
|         |                  |
|     One kernel thread      |
+---------------------------+
The OS only sees one thread. The runtime
manages the green threads internally.
```

Each approach has trade-offs:

- **Kernel threads**: The OS can schedule them across multiple cores (parallelism). But context switches are expensive because they involve the kernel.
- **User-level threads**: Context switches are very cheap (no kernel involvement). But the OS only sees one thread, so they all run on one core (no parallelism).

## Go's Approach: Goroutines

Go uses a clever **hybrid approach** called **M:N scheduling**. Go creates M goroutines (user-level) and maps them onto N OS threads (kernel-level). The Go runtime scheduler manages goroutines in user space, but it uses multiple OS threads so goroutines can run in parallel.

```
Go's M:N Thread Model:

+------------------------------------------+
|            Go Runtime                     |
|                                          |
|  Goroutines (M):                         |
|  G1  G2  G3  G4  G5  G6  G7  G8        |
|   |   |   |   |   |   |   |   |         |
|   v   v   v   v   v   v   v   v         |
|  +---------------------------+           |
|  |   Go Runtime Scheduler    |           |
|  +---------------------------+           |
|       |           |           |          |
|       v           v           v          |
|  OS Threads (N):                          |
|  Thread 1   Thread 2   Thread 3          |
+------------------------------------------+
       |           |           |
       v           v           v
  +------------------------------------+
  |          OS Kernel                  |
  |  Core 0    Core 1    Core 2        |
  +------------------------------------+
```

This gives Go the best of both worlds:
- **Cheap context switches** between goroutines (handled by the Go runtime in user space)
- **Real parallelism** because goroutines are scheduled across multiple OS threads
- **Efficient use of system resources** because thousands of goroutines share a few OS threads

## Thread Safety Issues

Because threads share memory, you get thread safety problems. When two threads access the same variable and at least one of them writes to it, you get a **data race**.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // This code has a data race!
    var counter int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // multiple goroutines writing to same variable
        }()
    }

    wg.Wait()
    // counter might not be 1000 due to data race
    fmt.Printf("Counter: %d (expected 1000)\n", counter)
}
```

The fix is to use synchronization. Go provides several tools for this:

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    // Fix 1: Use a mutex
    var counter1 int
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter1++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Printf("Mutex counter: %d\n", counter1)

    // Fix 2: Use atomic operations
    var counter2 int64
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter2, 1)
        }()
    }
    wg.Wait()
    fmt.Printf("Atomic counter: %d\n", counter2)

    // Fix 3: Use channels (Go's preferred way)
    ch := make(chan int, 1000)
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            ch <- 1
        }()
    }
    go func() {
        wg.Wait()
        close(ch)
    }()
    counter3 := 0
    for range ch {
        counter3++
    }
    fmt.Printf("Channel counter: %d\n", counter3)
}
```

## Why Go Uses Goroutines Instead of OS Threads

The table says it all:

| Aspect                | OS Thread          | Goroutine              |
|----------------------|--------------------|------------------------|
| Initial stack size    | 1-8 MB             | 2 KB                   |
| Creation cost        | ~1-10 microseconds | ~100-300 nanoseconds   |
| Context switch cost  | ~1-10 microseconds | ~100-200 nanoseconds   |
| Max practical count  | Thousands          | Hundreds of thousands  |
| Managed by           | OS kernel          | Go runtime             |
| Scheduling            | Preemptive (OS)    | Cooperative + preemptive|

You can easily create 100,000 goroutines in Go. Try creating 100,000 OS threads and your system will probably crash or slow to a crawl.

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    // How many goroutines can we create?
    const numGoroutines = 100_000

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Each goroutine just exists and waits
        }(i)
    }

    fmt.Printf("Created %d goroutines\n", numGoroutines)
    fmt.Printf("Currently running: %d goroutines\n", runtime.NumGoroutine())

    wg.Wait()
    fmt.Println("All goroutines finished")
}
```

This runs fine. A hundred thousand goroutines, no problem. Each one starts with just 2KB of stack. That is only about 200MB total, which is nothing on a modern machine. A hundred thousand OS threads at 1MB stack each would need 100GB. That is not going to work.

Goroutines are Go's superpower. They give you the simplicity of threads without the overhead. And understanding threads helps you appreciate why goroutines are so much better.
