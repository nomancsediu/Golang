# CPU and Processes

## What Is a Process?

When I first heard the word "process," I thought it was just another word for "program." But they are not the same thing. A **program** is a file on disk. A **process** is a running instance of that program.

Think of it this way: a program is like a recipe, and a process is like someone actually cooking from that recipe. You can have many people cooking from the same recipe at the same time, which means one program can have multiple processes running.

When you run `go run main.go`, the operating system creates a new process. It loads your program code into memory, sets up a stack and a heap, and starts executing instructions.

## What a Process Contains

Every process has several parts:

```
+----------------------------------+
|           Process                |
|                                  |
|  +----------------------------+  |
|  |  Code Segment (Text)       |  |
|  |  The compiled instructions |  |
|  |  (read-only, shared)       |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |  Data Segment              |  |
|  |  Global variables          |  |
|  |  Static variables          |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |  Heap                      |  |
|  |  Dynamically allocated     |  |
|  |  memory (grows upward)     |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |  Stack                     |  |
|  |  Local variables           |  |
|  |  Function call frames      |  |
|  |  (grows downward)          |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |  File Descriptors          |  |
|  |  Open files, sockets, etc. |  |
|  +----------------------------+  |
+----------------------------------+
```

- **Code segment** - The actual machine instructions of your program. This is read-only and can be shared between multiple processes running the same program.
- **Data segment** - Global and static variables. These exist for the entire lifetime of the process.
- **Heap** - Memory that is dynamically allocated during runtime (in Go, this is where the garbage collector manages memory).
- **Stack** - Local variables and function call information. Each function call pushes a new frame onto the stack.
- **File descriptors** - A table of open files, network connections, and other I/O resources. In Unix-like systems, everything is a file, so file descriptors are very important.

## CPU Executes One Instruction at a Time

This was a surprise to me. A single CPU core can only execute **one instruction at a time**. It fetches an instruction, decodes it, executes it, and then moves to the next one. That is it. One at a time.

But wait, I have dozens of programs running on my computer right now. My browser, my editor, my music player, my terminal. How can they all run at the same time if the CPU only does one thing at a time?

The answer is **time sharing**. The operating system rapidly switches between processes, giving each one a tiny slice of CPU time. It happens so fast (thousands of times per second) that it looks like everything is running simultaneously.

```
Time -------------------------------------------------->

Core 1:  [App A][App B][App C][App A][App B][App C]...
                ^                ^
                |                |
           App A gets      App A gets
           CPU time again  CPU time again
```

Each process gets a **time slice** (or quantum), typically around 1 to 10 milliseconds. When the time slice is up, the OS pauses the process and gives the CPU to another process. This happens so fast that we do not notice it.

## Process States

A process is not always running. It goes through different states during its lifetime:

```
                    +---------+
                    |  New    |  (process being created)
                    +---------+
                         |
                         v
                    +---------+
              +---->|  Ready  |<----+
              |     +---------+     |
              |          |          |
              |   scheduler        |
              |   picks it         |
              |          |          |
              |          v          |
              |     +---------+    |
              |     | Running |----+ time slice expired
              |     +---------+
              |          |
              |   needs I/O or
              |   waits for resource
              |          |
              |          v
              |     +---------+
              +-----| Waiting |
         I/O done   +---------+
              |
              v
         +-----------+
         | Terminated |  (process finished or killed)
         +-----------+
```

- **New** - The process is being created
- **Ready** - The process is waiting for CPU time (it could run, but the CPU is busy with something else)
- **Running** - The process is currently executing on the CPU
- **Waiting** - The process is waiting for something (I/O operation, lock, timer, etc.)
- **Terminated** - The process has finished or been killed

The key insight here: most processes spend a lot of time in the **Waiting** state. When a Go program reads from a file or waits for a network response, the process is waiting. The OS can give the CPU to another process during this time. This is why your computer feels responsive even with many programs running.

## What the OS Stores for Each Process

When the OS switches from one process to another, it needs to remember everything about the first process so it can resume it later. This is called the **process context**, and it includes:

- **Program counter** - which instruction the CPU was about to execute
- **Registers** - all CPU register values
- **Stack pointer** - where the stack currently is
- **Memory mapping** - which virtual memory pages belong to this process
- **Open file descriptors** - which files and sockets the process has open
- **Scheduling info** - priority, time slice remaining, etc.

All of this is stored in a data structure called the **Process Control Block (PCB)**, which we will cover in detail in a later section.

## Why This Matters for Go

Understanding processes helps you understand several things about Go:

1. **Go programs are processes** - When you run a Go binary, the OS creates a process with all the components described above.

2. **Go's runtime manages goroutines within one process** - Instead of creating a new OS process for each concurrent task, Go creates lightweight goroutines within a single process. This is much more efficient.

3. **Blocking I/O matters** - When a Go program makes a blocking I/O call, the process (or thread) goes into the Waiting state. The Go runtime handles this by moving goroutines around so other goroutines can keep running.

4. **Why programs feel slow** - If your system has too many processes, the OS spends more time switching between them and less time actually doing useful work. This is called thrashing.

## Seeing Process Info in Go

You can look at process information from within a Go program:

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "runtime"
)

func main() {
    // Get the current process ID
    pid := os.Getpid()
    fmt.Printf("Process ID: %d\n", pid)

    // Get the parent process ID
    ppid := os.Getppid()
    fmt.Printf("Parent Process ID: %d\n", ppid)

    // Number of CPU cores available
    fmt.Printf("CPU cores: %d\n", runtime.NumCPU())

    // Memory stats for this Go process
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Heap allocated: %d KB\n", m.HeapAlloc/1024)
    fmt.Printf("Heap sys (from OS): %d KB\n", m.HeapSys/1024)
    fmt.Printf("Total sys: %d KB\n", m.Sys/1024)
    fmt.Printf("Num goroutines: %d\n", runtime.NumGoroutine())

    // You can also start new processes
    cmd := exec.Command("echo", "hello from child process")
    output, err := cmd.Output()
    if err != nil {
        fmt.Println("Error running command:", err)
        return
    }
    fmt.Printf("Child process output: %s", output)
}
```

When I run this, I can see my Go process has its own ID, it knows how many CPU cores are available, and it can report on its own memory usage. The Go runtime gives us a lot of visibility into what is happening inside our process.

Understanding processes is the first step toward understanding concurrency. Next, we will look at what happens when the OS switches between processes.
