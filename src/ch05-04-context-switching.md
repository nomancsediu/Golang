# Context Switching

## The Hidden Cost of Multitasking

I used to think multitasking was free. You just run multiple programs at the same time and everything works. But there is a hidden cost every time the operating system switches from one process to another. This cost is called a **context switch**, and understanding it completely changed how I think about Go's concurrency model.

## What Is a Context Switch?

A **context switch** is when the operating system stops running one process and starts running another. To do this, the OS has to:

1. **Save** the current state of the running process (so it can be resumed later)
2. **Load** the saved state of the next process (so it can continue where it left off)

Think of it like reading two books at the same time. You are reading Book A, and then you need to switch to Book B. Before you switch, you put a bookmark in Book A to remember where you were. Then you find the bookmark in Book B and start reading from there. The act of putting in the bookmark and finding the other bookmark is the context switch.

## What Gets Saved and Loaded

During a context switch, the OS has to save and restore a lot of state:

```
+-----------------------------------------+
|         Saved During Context Switch      |
|                                         |
|  +-----------------------------------+  |
|  |  General Purpose Registers         |  |
|  |  (RAX, RBX, RCX, RDX, RSI, etc.)  |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Program Counter (RIP)            |  |
|  |  (address of next instruction)    |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Stack Pointer (RSP)              |  |
|  |  (current position in stack)      |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Frame Pointer (RBP)              |  |
|  |  (base of current stack frame)    |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Floating Point / Vector regs     |  |
|  |  (SIMD, FPU state)                |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Memory Mapping (page tables)     |  |
|  |  (virtual to physical mapping)    |  |
|  +-----------------------------------+  |
|                                         |
|  +-----------------------------------+  |
|  |  Other: flags, TLS, signal masks  |  |
|  +-----------------------------------+  |
+-----------------------------------------+
```

That is a lot of data to save and restore. On modern x86-64 processors, there are 16 general-purpose registers, plus floating-point registers, plus various control registers. All of this needs to be copied to memory (the PCB) and then loaded back.

## Context Switching Is Expensive

Here is the part that really surprised me: **a context switch takes between 1 and 10 microseconds on a typical system**. That might sound fast, but consider this:

- A CPU can execute billions of instructions per second
- During a 5 microsecond context switch, the CPU does no useful work for your program
- That is potentially millions of instructions wasted

And it gets worse. When the OS switches to a new process:

1. The **CPU cache** is now filled with data from the old process, not the new one
2. The **TLB** (Translation Lookaside Buffer, which caches virtual-to-physical memory mappings) needs to be flushed
3. The new process will have **cache misses** until the cache fills with its data

This means the real cost of a context switch is not just the switch itself, but also the **cache warm-up period** that follows. Some studies estimate the total cost at 10 to 30 microseconds or more when you include cache effects.

```
Timeline of a context switch:

Process A running     Context switch     Process B warming cache     Process B running
[====useful====] | [==save/load==] | [==cache misses==] | [====useful====]
                  ^                 ^
                  |                 |
            No useful work    Still slow due to
            during switch    cache misses
```

## Why Too Many Processes = Slow System

If you have 100 processes that all want CPU time, and the OS switches between them rapidly, a significant portion of CPU time is spent just switching between processes instead of doing actual work.

For example, if the time slice is 10 milliseconds and the context switch takes 20 microseconds:

- Time doing useful work: 10 ms
- Time spent switching: 0.02 ms
- Overhead: 0.2%

That does not seem bad. But if the time slice is reduced to 1 millisecond (which happens when there are many processes):

- Time doing useful work: 1 ms
- Time spent switching: 0.02 ms
- Overhead: 2%

And if you include cache effects, the overhead can be much higher. This is why systems with too many active processes start to feel slow. The OS is spending more time juggling processes than actually running them.

## How Go Reduces Context Switching

This is where Go really shines. Go uses **goroutines** instead of OS threads for concurrency, and goroutines have much cheaper context switches.

Here is why:

| Aspect              | OS Thread Context Switch | Goroutine Context Switch |
|--------------------|------------------------|-------------------------|
| What gets saved     | All registers, FPU, SIMD, page tables | Only 3 registers: SP, PC, DX |
| Time               | ~1-10 microseconds      | ~100-200 nanoseconds     |
| Cache impact        | High (full cache flush)  | Low (same address space)  |
| TLB flush           | Yes                      | No                        |
| Involves OS kernel  | Yes                      | No (managed by Go runtime)|

A goroutine context switch is about **10 to 100 times cheaper** than an OS thread context switch. This is why Go can efficiently handle hundreds of thousands of goroutines while a system would struggle with a few thousand OS threads.

The Go runtime manages goroutine switching entirely in user space. It does not need to involve the OS kernel for most switches, which avoids the expensive kernel transition.

## Code Example: Seeing the Difference

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func main() {
    // Measure goroutine context switch cost
    const iterations = 100000
    var wg sync.WaitGroup

    // Channel-based synchronization to force context switches
    ch1 := make(chan struct{})
    ch2 := make(chan struct{})

    start := time.Now()

    // Goroutine 1: ping
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < iterations; i++ {
            ch1 <- struct{}{}
            <-ch2
        }
    }()

    // Goroutine 2: pong
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < iterations; i++ {
            <-ch1
            ch2 <- struct{}{}
        }
    }()

    wg.Wait()
    elapsed := time.Since(start)

    switchesPerSec := float64(iterations*2) / elapsed.Seconds()
    nsPerSwitch := float64(elapsed.Nanoseconds()) / float64(iterations*2)

    fmt.Printf("Goroutine context switches: %d\n", iterations*2)
    fmt.Printf("Total time: %v\n", elapsed)
    fmt.Printf("Switches per second: %.0f\n", switchesPerSec)
    fmt.Printf("Nanoseconds per switch: %.1f\n", nsPerSwitch)
    fmt.Printf("Number of CPU cores: %d\n", runtime.NumCPU())
}
```

When I run this, I can see that goroutine switches take just a few hundred nanoseconds each. That is incredibly fast compared to OS thread switches.

## Visual Diagram of a Context Switch

```
            Time -->
            
CPU Core   [Process A] [  switch  ] [Process B]
                        |          |
                        v          v
                   +----------------------------+
                   | Save Process A state:      |
                   |   RIP = 0x7fff1234         |
                   |   RSP = 0x7fff8000         |
                   |   RAX = 42                 |
                   |   ...                      |
                   +----------------------------+
                   | Load Process B state:      |
                   |   RIP = 0x7ffe5678         |
                   |   RSP = 0x7ffe2000         |
                   |   RAX = 7                  |
                   |   ...                      |
                   +----------------------------+
                   | Flush TLB entries          |
                   | Cache now has A's data     |
                   | B will have cache misses   |
                   +----------------------------+
```

## The Takeaway

Context switching is not free. Every time the OS switches between processes or threads, there is a cost. Go's genius is in making the most common context switches (between goroutines) very cheap by managing them in user space instead of going through the OS kernel.

When you write `go myFunction()`, you are creating a goroutine that can be switched in and out in nanoseconds, not microseconds. That is why Go can handle so much concurrency so efficiently.
