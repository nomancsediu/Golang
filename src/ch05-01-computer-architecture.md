# Computer Architecture Basics

## The Machine Underneath

When I write `fmt.Println("hello")` in Go, something happens between me typing that and "hello" appearing on screen. The CPU has to execute instructions, memory has to be read and written, and data has to move around. Understanding this process changed how I think about code.

Let me walk through the basics of computer architecture. I promise it is not as scary as it sounds.

## The Big Three: CPU, RAM, and Storage

Every computer has three main components that matter to us as programmers:

**CPU (Central Processing Unit)** - The brain. It executes instructions one at a time (per core). It is fast. Very fast. But it can only work with data that is close to it.

**RAM (Random Access Memory)** - The workspace. This is where your running program lives. The CPU can read from and write to RAM, but it is much slower than the CPU itself.

**Storage (Disk/SSD)** - The filing cabinet. This is where files and data persist when the computer is off. It is way slower than RAM.

The key insight is: **there is a huge speed gap between these components**. The CPU is much faster than RAM, and RAM is much faster than disk.

## The Memory Hierarchy

To deal with the speed gap, computers have a memory hierarchy. Data that is used often is kept closer to the CPU, where it can be accessed faster:

```
+--------------------------------------------------+
|  Fastest / Smallest / Most Expensive              |
|                                                    |
|  Registers    (inside CPU, bytes)        ~0 ns     |
|  L1 Cache     (per core, ~32-64 KB)     ~1 ns     |
|  L2 Cache     (per core, ~256-512 KB)   ~4 ns     |
|  L3 Cache     (shared, ~4-32 MB)        ~10 ns    |
|  RAM          (shared, ~8-64 GB)        ~100 ns   |
|  SSD Disk     (shared, ~256 GB-TB)      ~100 μs   |
|  HDD Disk     (shared, ~1+ TB)          ~10 ms    |
|                                                    |
|  Slowest / Largest / Cheapest                     |
+--------------------------------------------------+
```

Notice the numbers on the right. Reading from L1 cache is about 100 times faster than reading from RAM. Reading from RAM is about 1000 times faster than reading from SSD. These differences are huge.

**Why this matters for programming**: If your code accesses memory in a way that keeps data in cache, it will be dramatically faster than code that keeps going back to RAM. This is called **cache-friendly code**.

## How Programs Run

Here is the simplified version of what happens when you run a Go program:

1. The **operating system** loads your program from disk into RAM
2. The **CPU** fetches the first instruction from RAM
3. The CPU **decodes** the instruction (figures out what to do)
4. The CPU **executes** the instruction (does the thing)
5. The CPU moves to the next instruction and repeats

This is called the **fetch-decode-execute cycle**, and it happens billions of times per second.

```
+----------+     fetch      +-----+
|          | -------------> |     |
|   RAM    |                | CPU |
|          | <------------- |     |
+----------+   execute      +-----+
     ^                         |
     |                         | decode
     +-------------------------+
              store result
```

## The Bus

Components talk to each other through something called a **bus**. Think of it as a highway for data. There are different buses:

- **Data bus** - carries the actual data between CPU, RAM, and other components
- **Address bus** - carries the memory address that the CPU wants to read from or write to
- **Control bus** - carries signals about what operation to perform (read, write, etc.)

When the CPU wants to read from memory address `0x1000`, it puts `0x1000` on the address bus, puts "read" on the control bus, and then the data comes back on the data bus.

## Von Neumann Architecture

Most computers follow the **Von Neumann architecture**, which has one key idea: **both data and instructions are stored in the same memory (RAM)**. This means the CPU treats code and data the same way. It is a simple but powerful design.

```
+-----------------------------------+
|            Memory (RAM)           |
|  +-------+  +-------+  +-------+ |
|  |Instruction| |Instruction| |Data | |
|  |   0x0000  | |  0x0001  | |0x0002| |
|  +-------+  +-------+  +-------+ |
|  +-------+  +-------+  +-------+ |
|  |  Data |  |  Data |  |  Data | |
|  | 0x0003 | | 0x0004 | | 0x0005 | |
|  +-------+  +-------+  +-------+ |
+-----------------------------------+
         ^                  |
         |  fetch/execute   |
         v                  |
    +----------+            |
    |   CPU    |            |
    +----------+            |
         ^                  |
         |    I/O devices   |
         v                  |
    +----------+            |
    |  Input/  | <---------+
    |  Output  |
    +----------+
```

The limitation of Von Neumann architecture is the **Von Neumann bottleneck**: the CPU and RAM share a single bus, so instructions and data cannot be fetched at the same time. The CPU often has to wait for data to arrive.

## Cache-Friendly vs Cache-Unfriendly Code

This is where architecture meets programming. Let me show you a real example in Go.

When you access memory sequentially (row by row in a 2D array), the CPU can prefetch the next chunk of data into cache. When you access memory randomly or in a "wrong" order (column by column), the CPU keeps missing the cache and going back to RAM.

```go
package main

import (
    "fmt"
    "time"
)

const size = 4096

func main() {
    // Create a 2D array (matrix)
    matrix := make([][]int, size)
    for i := range matrix {
        matrix[i] = make([]int, size)
    }

    // Fill with some values
    for i := 0; i < size; i++ {
        for j := 0; j < size; j++ {
            matrix[i][j] = i + j
        }
    }

    // Cache-friendly: row by row (sequential access)
    start := time.Now()
    sum := 0
    for i := 0; i < size; i++ {
        for j := 0; j < size; j++ {
            sum += matrix[i][j] // rows are contiguous in memory
        }
    }
    fmt.Printf("Row-by-row: %v (sum=%d)\n", time.Since(start), sum)

    // Cache-unfriendly: column by column (strided access)
    start = time.Now()
    sum = 0
    for j := 0; j < size; j++ {
        for i := 0; i < size; i++ {
            sum += matrix[i][j] // columns skip across memory
        }
    }
    fmt.Printf("Column-by-column: %v (sum=%d)\n", time.Since(start), sum)
}
```

When I ran this on my machine, the row-by-row version was about 3 to 5 times faster. Same data, same computation, different access pattern. That is the power of understanding the memory hierarchy.

**The takeaway**: Memory is not just memory. How you access it matters a lot. Sequential access is fast because the CPU cache can predict and preload the next chunk of data. Random or strided access is slow because the cache keeps missing.

## Why This Matters for Go Developers

- **Slice layout**: Go slices are stored as contiguous memory blocks. Iterating over a slice sequentially is cache-friendly.
- **Struct layout**: The order of fields in a struct can affect cache performance. Fields accessed together should be placed together.
- **Heap vs stack**: Stack data is more cache-friendly because it is contiguous. Heap data can be scattered around memory.
- **Data-oriented design**: Thinking about how your data is laid out in memory can make your Go programs significantly faster.

Understanding the hardware does not make you a systems programmer. It makes you a better programmer, period.
