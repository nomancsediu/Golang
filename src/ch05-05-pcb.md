# PCB (Process Control Block)

## How the OS Remembers Everything About a Process

When I first learned that the OS needs to store information about every process, I wondered: where does it put all that data? The answer is the **Process Control Block**, or PCB. Every process has one, and the OS uses it to keep track of everything about that process.

Think of the PCB like a file folder for each process. Whenever the OS needs to know something about a process, or needs to save the current state of a process, it uses the PCB.

## What Is a PCB?

The **Process Control Block** is a data structure that the operating system uses to store all the information it needs about a process. It is also sometimes called a **process descriptor** or **task struct** (in Linux, it is called `task_struct`).

Every process has exactly one PCB. The PCB is created when the process is created and is destroyed when the process terminates. The OS maintains a collection of all PCBs, often organized as a list or queue for scheduling purposes.

## What Does the PCB Contain?

The PCB stores a lot of information. Here is a breakdown:

```
+=============================================+
|          Process Control Block (PCB)         |
+=============================================+
|                                             |
|  PROCESS IDENTIFICATION                     |
|  +---------------------------------------+  |
|  | Process ID (PID)                      |  |
|  | Parent Process ID (PPID)              |  |
|  | User ID (UID)                         |  |
|  | Group ID (GID)                        |  |
|  +---------------------------------------+  |
|                                             |
|  PROCESS STATE                              |
|  +---------------------------------------+  |
|  | Current state (new/ready/running/     |  |
|  |   waiting/terminated)                 |  |
|  | Priority                              |  |
|  | Scheduling queue pointers             |  |
|  | Time slice remaining                  |  |
|  | Total CPU time used                   |  |
|  | Time started                          |  |
|  +---------------------------------------+  |
|                                             |
|  CPU REGISTERS (saved during context switch)|
|  +---------------------------------------+  |
|  | Program Counter (PC)                  |  |
|  | Stack Pointer (SP)                    |  |
|  | General purpose registers             |  |
|  | Floating point registers              |  |
|  | Condition codes / flags               |  |
|  +---------------------------------------+  |
|                                             |
|  MEMORY MANAGEMENT                          |
|  +---------------------------------------+  |
|  | Page table base register              |  |
|  | Memory limits (start, size)           |  |
|  | Segmentation info                     |  |
|  | Shared memory regions                 |  |
|  +---------------------------------------+  |
|                                             |
|  I/O STATUS                                 |
|  +---------------------------------------+  |
|  | List of open file descriptors         |  |
|  | I/O devices allocated                 |  |
|  | Current working directory             |  |
|  | File creation mask (umask)            |  |
|  +---------------------------------------+  |
|                                             |
|  ACCOUNTING                                 |
|  +---------------------------------------+  |
|  | CPU time used (user)                  |  |
|  | CPU time used (system)                |  |
|  | Page faults                           |  |
|  | Signals pending                       |  |
|  +---------------------------------------+  |
|                                             |
+=============================================+
```

That is a lot of information for just one process. And the OS has to manage PCBs for every single process on the system. If you have 500 processes running, the OS is maintaining 500 PCBs.

## PCB Lifecycle

The PCB follows the process through its entire life:

```
Process Created
      |
      v
+-----------+     Scheduler      +-----------+
| PCB       |  picks process     | PCB       |
| State:    | --------------->   | State:    |
|  READY    |                    | RUNNING   |
+-----------+                    +-----------+
      ^                               |
      |                               |
      | time slice              needs |
      | expired                 I/O   |
      |                               v
      |                        +-----------+
      +------------------------| PCB       |
       I/O done                | State:    |
                               | WAITING   |
                               +-----------+
                                        |
                                        | process
                                        | exits
                                        v
                                 +-----------+
                                 | PCB       |
                                 | State:    |
                                 | TERMINATED|
                                 +-----------+
                                        |
                                        | resources
                                        | freed
                                        v
                                   PCB removed
```

1. **Process is created** - The OS allocates a new PCB and fills in the initial values
2. **Process is ready** - The PCB is in the ready queue, waiting for CPU time
3. **Process is running** - The PCB is updated with current register values when context switches happen
4. **Process is waiting** - The PCB records what the process is waiting for (I/O, signal, etc.)
5. **Process terminates** - The PCB is used to clean up resources, then the PCB itself is removed

## How the OS Uses PCBs for Scheduling

The OS maintains several **queues** of PCBs:

- **Ready queue** - PCBs of processes that are ready to run but waiting for CPU time
- **Wait queues** - PCBs of processes waiting for specific events (one queue per event type, like "waiting for disk I/O")
- **Job queue** - PCBs of all processes in the system

When the scheduler needs to pick a process to run, it looks at the ready queue and selects a PCB based on the scheduling algorithm. When a process starts waiting for I/O, its PCB moves from the ready queue to the appropriate wait queue. When the I/O completes, the PCB moves back to the ready queue.

```
         Ready Queue
   +------+------+------+------+
   | PCB1 | PCB2 | PCB3 | PCB4 |
   +------+------+------+------+
      |                    
      | scheduler picks
      v
   +------+     Running
   | PCB2 | --------------> CPU
   +------+
      |
      | needs I/O
      v
   I/O Wait Queue
   +------+------+------+
   | PCB2 | PCB5 | PCB7 |
   +------+------+------+
```

## PCBs in the Real World: Linux task_struct

In Linux, the PCB is implemented as a C struct called `task_struct`. It is defined in the Linux kernel source and contains over 300 fields. Here is a tiny simplified version:

```c
// Simplified version of Linux task_struct
struct task_struct {
    long state;            // process state
    int prio;              // priority
    pid_t pid;             // process ID
    pid_t tgid;            // thread group ID
    struct mm_struct *mm;  // memory management
    struct files_struct *files;  // open files
    struct signal_struct *signal; // signal handling
    struct thread_info thread_info; // low-level CPU state
    // ... 300+ more fields
};
```

Every process and thread in Linux has a `task_struct`. Yes, in Linux, threads are also represented by `task_struct` objects. They just share some resources (like memory space) with other threads in the same process.

## Go's Runtime Maintains Similar Structures for Goroutines

Go does not use PCBs directly (that is the OS's job), but the Go runtime maintains a very similar data structure for each goroutine. In the Go source code, this is called the **`g` struct** (for "goroutine").

Here is a simplified view of what Go stores for each goroutine:

```go
// Simplified version of Go's internal g struct
type g struct {
    stack       stack    // stack bounds [lo, hi]
    sched       gobuf    // saved registers for context switch
    atomicstatus uint32  // goroutine state
    goid        uint64   // goroutine ID
    waitsince   int64    // time when started waiting
    waitreason  string   // why the goroutine is waiting
    m           *m       // which OS thread is running this goroutine
    // ... many more fields
}

// Saved context for goroutine switches
type gobuf struct {
    sp   uintptr  // stack pointer
    pc   uintptr  // program counter
    g    guintptr // goroutine pointer
    ret  uintptr  // return value
    // ... a few more fields
}
```

Notice how much simpler the `gobuf` is compared to a full OS context. Go only needs to save a few values (stack pointer, program counter, and a couple others) because all goroutines share the same address space.

## Why This Matters

Understanding PCBs helps you understand:

1. **Why process creation is expensive** - The OS has to allocate and initialize a whole PCB, set up memory, copy file descriptors, etc. This is why `os.StartProcess()` in Go is slow compared to `go func()`.

2. **Why context switching is expensive** - The OS has to save and restore all the PCB data, including registers and memory mappings. The PCB makes this possible but also makes it costly.

3. **Why goroutines are cheaper** - Go's `g` struct is much simpler than an OS PCB. Goroutines share memory, file descriptors, and other resources, so there is less to save and restore.

4. **How the scheduler works** - The OS uses PCBs to decide which process to run next. The Go runtime uses its own structures to decide which goroutine to run next. Same concept, different scale.

The PCB is the OS's way of keeping track of processes. Go's runtime does something similar for goroutines, but in a much lighter way. That lightness is a big part of why Go concurrency is so efficient.
