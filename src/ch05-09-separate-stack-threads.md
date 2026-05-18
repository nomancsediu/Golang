# Separate Stack for Separate Thread

## Why Every Thread Needs Its Own Stack

This topic seems small, but it is actually one of the most important things I learned about why goroutines are so efficient. Understanding stacks changed my entire mental model of how Go achieves its legendary concurrency.

## What Is a Stack?

A **stack** is a region of memory that a thread (or goroutine) uses to store:

- **Local variables** - variables declared inside a function
- **Function call frames** - information about each active function call
- **Return addresses** - where to go back to after a function finishes
- **Function arguments** - values passed to functions
- **Temporary values** - intermediate computation results

Every thread needs its own stack because each thread has its own independent execution flow. Two threads might be calling the same function at the same time, but they each need their own copy of local variables and return addresses.

```
Thread A's Stack         Thread B's Stack
+------------------+    +------------------+
| main() frame     |    | main() frame     |
|  - return addr   |    |  - return addr   |
|  - local vars    |    |  - local vars    |
+------------------+    +------------------+
| processData()    |    | handleRequest()  |
|  - return addr   |    |  - return addr   |
|  - local vars    |    |  - local vars    |
+------------------+    +------------------+
| computeValue()   |    | validateInput()  |
|  - return addr   |    |  - return addr   |
|  - x = 42        |    |  - ok = true     |
|  - y = "hello"   |    |  - err = nil     |
+------------------+    +------------------+
| (free space)     |    | (free space)     |
| ...              |    | ...              |
+------------------+    +------------------+
```

If two threads shared the same stack, they would overwrite each other's local variables and return addresses. That would be chaos. Each thread must have its own stack to maintain an independent execution flow.

## OS Thread Stack Size

On most operating systems, each OS thread gets a fixed stack size:

| Operating System | Default Stack Size |
|-----------------|-------------------|
| Linux           | 8 MB              |
| macOS           | 512 KB (main), 8 MB (others) |
| Windows         | 1 MB              |

That is a lot of memory per thread. Even if a thread only uses a few kilobytes of stack, the OS allocates the full amount. Most of that space is wasted.

Let me do the math. If you create 10,000 OS threads on Linux:

```
10,000 threads x 8 MB per thread = 80,000 MB = 80 GB
```

That is 80 gigabytes just for stacks. No normal computer has that much RAM. This is why creating thousands of OS threads is impractical.

Even with a smaller stack size (some systems allow you to configure it), OS thread stacks are relatively large because:

1. They cannot easily grow (fixed size)
2. They need to handle the worst case (deeply nested function calls)
3. Growing a stack in an OS thread requires remapping memory, which is expensive

## Goroutine Stack Size: Start Small, Grow As Needed

This is where Go is brilliant. **Goroutines start with a tiny stack of just 2 KB**. Not 8 MB. Not 1 MB. Just 2 kilobytes. That is 4000 times smaller than a typical OS thread stack.

But what if a goroutine needs more stack space? Go handles this automatically. When a goroutine's stack runs out of space, the Go runtime **grows the stack**. This is transparent to the programmer.

```
Goroutine Stack Growth:

Initial: 2 KB                    After growth: 4 KB              After growth: 8 KB
+------------------+            +------------------+            +------------------+
| current frames   |            | current frames   |            | current frames   |
+------------------+            |                  |            |                  |
| (free space)     |            | more frames      |            | more frames      |
+------------------+            | added during     |            | added during     |
                                | growth           |            | growth           |
                                +------------------+            |                  |
                                                                | even more frames |
                                                                +------------------+
                                                                | (free space)     |
                                                                +------------------+
```

The math now works out. If you create 100,000 goroutines:

```
100,000 goroutines x 2 KB = 200,000 KB = 200 MB
```

200 megabytes is totally manageable. And most goroutines will not need more than their initial 2 KB. Only goroutines with deeply nested calls or large local variables will grow their stacks.

## What the Stack Stores

Let me look at exactly what goes on the stack during a function call in Go:

```go
package main

import "fmt"

func main() {
    x := 10           // local variable on main's stack frame
    result := add(x, 5) // function call creates new frame
    fmt.Println(result)
}

func add(a int, b int) int {  // a and b are on add's stack frame
    sum := a + b              // local variable on add's stack frame
    return sum
}
```

Here is what the stack looks like when `add` is executing:

```
Stack (growing downward)
+---------------------------+
| main() stack frame        |
|   x = 10                  |
|   result = (pending)      |
|   return address          |
+---------------------------+
| add() stack frame         |
|   a = 10                  |
|   b = 5                   |
|   sum = 15                |
|   return address          |
+---------------------------+
| (free space)              |
| ...                       |
+---------------------------+
```

Each function call pushes a new frame onto the stack. When the function returns, the frame is popped off. This is why stack allocation is so fast: you just move the stack pointer.

## Stack Overflow

A **stack overflow** happens when the stack runs out of space. This usually occurs with:

- **Deeply recursive functions** that never terminate
- **Extremely deep call chains**
- **Very large local variables** on the stack

In an OS thread with a fixed stack, hitting the stack limit causes a crash (stack overflow error, often a segmentation fault).

In Go, the runtime grows the stack automatically to prevent overflow. But there is still a maximum: Go goroutines have a default maximum stack size of 1 GB (as of Go 1.11). If a goroutine's stack exceeds this, the program crashes.

```go
package main

import "fmt"

// A deeply recursive function to test stack growth
func recurse(depth int) int {
    if depth <= 0 {
        return 0
    }
    // Create a local variable to use stack space
    // Each frame uses some bytes, so deep recursion
    // forces the stack to grow
    var buf [1024]byte // 1 KB per frame
    buf[0] = byte(depth)
    return recurse(depth-1) + int(buf[0])
}

func main() {
    // This will grow the goroutine's stack significantly
    result := recurse(10000)
    fmt.Printf("Result: %d\n", result)
    // The goroutine's stack grew from 2KB to handle
    // 10000 nested calls, each using ~1KB
}
```

Go handles this gracefully. The stack grows as needed. In the old days (before Go 1.3), Go used **segmented stacks** (also called split stacks), where the stack was a linked list of segments. This had a problem called the "hot split" issue: if a function call was right at the boundary of a stack segment, the runtime would constantly allocate and free segments.

Since Go 1.3, Go uses **contiguous stacks** (also called copying stacks). When the stack needs to grow, the runtime allocates a new, larger stack and copies the old stack data to the new one. This is more efficient and avoids the hot split problem.

```
Contiguous Stack Growth:

1. Old stack (2 KB) is running low on space
2. Allocate new stack (4 KB)
3. Copy all data from old stack to new stack
4. Update all pointers that pointed into old stack
5. Free old stack
6. Continue execution on new stack

+--------+        +----------------+
| Old    |  copy  | New            |
| Stack  | -----> | Stack          |
| 2 KB   |        | 4 KB           |
+--------+        +----------------+
    |                     |
    v                     v
  freed               now in use
```

The pointer updating in step 4 is the tricky part. Go has to find every pointer that referenced the old stack and update it to point to the corresponding location in the new stack. This is possible because Go is a garbage-collected language and the runtime knows the layout of all data on the stack.

## Visual Comparison: OS Thread vs Goroutine Stacks

```
OS Thread Stacks (10 threads, 8MB each):

+-------------------------------------------+
| Thread 1  | Thread 2  | Thread 3  | ...   |
| 8 MB      | 8 MB      | 8 MB      |       |
| [used][---free---]  [used][--free--]       |
| Total: 80 MB for 10 threads               |
+-------------------------------------------+

Goroutine Stacks (10,000 goroutines, 2KB each):

+--+--+--+--+--+--+--+--+--+--+--+--+--+
|G1|G2|G3|G4|G5|G6|G7|G8|G9|..|..|..|..|
|2K|2K|4K|2K|2K|2K|8K|2K|2K|..|..|..|..|
+--+--+--+--+--+--+--+--+--+--+--+--+--+
| Total: ~20-30 MB for 10,000 goroutines |
+---------------------------------------+
(only goroutines that need more space grow)
```

Notice how most goroutines stay at 2 KB, and only a few grow larger. This is the key insight. Most goroutines do simple work and never need a big stack. Go only allocates more memory for the goroutines that actually need it.

## Code Example: Observing Goroutine Stack Growth

```go
package main

import (
    "fmt"
    "runtime"
)

func growStack(depth int) {
    if depth <= 0 {
        return
    }
    // Local array to consume stack space
    // 8 KB per frame (1024 int64 values)
    var buf [1024]int64
    buf[0] = int64(depth)
    growStack(depth - 1)
}

func main() {
    // Print initial goroutine stack info
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    fmt.Printf("Before: HeapAlloc = %d KB\n", memStats.HeapAlloc/1024)

    // Start a goroutine that will need a large stack
    done := make(chan struct{})
    go func() {
        growStack(100)
        close(done)
    }()

    <-done

    runtime.ReadMemStats(&memStats)
    fmt.Printf("After:  HeapAlloc = %d KB\n", memStats.HeapAlloc/1024)
    fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())
}
```

You cannot directly measure a goroutine's stack size from Go code (the runtime does not expose this), but you can see the memory impact of stack growth through the memory statistics.

## Why This Matters

Understanding stacks is key to understanding Go's concurrency advantage:

1. **OS thread stacks are huge** (1-8 MB each), so you can only have a few thousand threads before running out of memory.

2. **Goroutine stacks start tiny** (2 KB each), so you can have hundreds of thousands of goroutines with modest memory use.

3. **Goroutine stacks grow automatically** when needed, so you get safety without waste.

4. **Contiguous stacks** (since Go 1.3) make growth efficient by copying to a larger allocation.

This is the fundamental reason why Go can handle massive concurrency that would overwhelm thread-based systems. It is not magic. It is just smart stack management. And now that I understand how it works, I can make better decisions about when and how to use goroutines in my programs.
