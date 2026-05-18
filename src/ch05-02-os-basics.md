# Operating System Basics

## What Does the OS Actually Do?

For a long time, I thought the operating system was just the thing that showed me a desktop and let me open apps. But the OS does so much more than that. It is the middleman between your programs and the hardware, and understanding what it does changed how I think about Go code.

The **operating system** is a special program that manages all the hardware and software on your computer. It provides a layer of abstraction so your Go program does not need to know the details of every hardware device.

Here is what the OS does for us:

1. **Manages hardware** - CPU, memory, disk, network, keyboard, screen
2. **Runs programs** - loads them, gives them CPU time, manages their lifecycle
3. **Provides abstractions** - files, sockets, processes, threads
4. **Protects programs from each other** - each program gets its own memory space
5. **Provides APIs** - system calls that programs use to request services

## Kernel vs User Space

This was a big "aha" moment for me. The CPU has different privilege levels. The OS kernel runs at the highest privilege level, and regular programs run at a lower privilege level.

**Kernel space** is where the OS kernel runs. It has full access to all hardware and memory. The kernel can do anything: read/write any memory, talk to any device, manage all processes.

**User space** is where your Go programs run. They have restricted access. They cannot directly access hardware, cannot read other programs' memory, and cannot do anything the kernel has not allowed.

```
+-----------------------------------------------+
|                  User Space                    |
|                                                |
|  +----------+  +----------+  +----------+     |
|  | Go prog  |  | Go prog  |  | Go prog  |     |
|  |   (app)  |  |   (app)  |  |   (app)  |     |
|  +----------+  +----------+  +----------+     |
|        |             |             |           |
|     syscall       syscall       syscall        |
|        |             |             |           |
+--------|-------------|-------------|-----------+
         v             v             v
+-----------------------------------------------+
|                Kernel Space                     |
|                                                |
|  +---------+  +----------+  +----------+      |
|  | Process |  | Memory   |  | File     |      |
|  | Manager |  | Manager  |  | System   |      |
|  +---------+  +----------+  +----------+      |
|  +---------+  +----------+                    |
|  | Network |  | Device   |                    |
|  | Stack   |  | Drivers  |                    |
|  +---------+  +----------+                    |
+-----------------------------------------------+
         |
         v
+-----------------------------------------------+
|              Hardware                           |
|  CPU    RAM    Disk    Network    Devices       |
+-----------------------------------------------+
```

When your Go program wants to do something like read a file or send network data, it cannot just do it directly. It has to ask the kernel to do it. This is called a **system call**.

## System Calls

A **system call** (or syscall) is how a user program asks the kernel to do something on its behalf. It is like knocking on the kernel's door and saying "hey, can you read this file for me?"

Common syscalls that Go programs use all the time:

| Syscall      | What it does                    | Go function that uses it     |
|-------------|--------------------------------|------------------------------|
| `read`       | Read data from a file/socket   | `os.File.Read()`, `net.Conn.Read()` |
| `write`      | Write data to a file/socket    | `os.File.Write()`, `net.Conn.Write()` |
| `open`       | Open a file                    | `os.Open()`                  |
| `close`      | Close a file descriptor        | `file.Close()`               |
| `socket`     | Create a network socket        | `net.Listen()`, `net.Dial()` |
| `fork`       | Create a new process           | `os.StartProcess()`          |
| `mmap`       | Map file into memory           | `syscall.Mmap()`             |
| `exit`       | Terminate the process          | `os.Exit()`                  |

Every time you open a file, read from a network connection, or create a new process in Go, a system call happens. Your program transitions from user space to kernel space, the kernel does the work, and then it transitions back.

This transition is not free. It takes time. Each syscall has overhead:

1. Save the user program state
2. Switch to kernel mode
3. The kernel does the work
4. Switch back to user mode
5. Restore the user program state

That is why, for example, reading a file byte by byte is much slower than reading it in chunks. Each read is a syscall, and each syscall has overhead.

## What the OS Provides

Let me go through each major thing the OS provides and why it matters for Go developers.

### Process Management

The OS creates, schedules, and terminates processes. When you run `go run main.go`, the OS creates a new process. The OS decides when your process gets CPU time and for how long.

### Memory Management

The OS gives each process its own **virtual memory** space. Your Go program thinks it has a big contiguous block of memory, but the OS is actually mapping that to physical RAM behind the scenes. The OS also handles **paging** (swapping memory to disk when RAM is full).

### File System

The OS provides the abstraction of files and directories. Your Go program does not need to know whether data is on an SSD, HDD, or network drive. It just opens a file path and reads/writes.

### Networking

The OS provides the network stack (TCP/IP). When your Go program calls `net.Dial()` or `net.Listen()`, it is asking the OS to create sockets, establish connections, and send/receive data.

## Seeing Syscalls in Go

You can actually see the syscalls your Go program makes. Here is a simple example:

```go
package main

import (
    "fmt"
    "os"
    "syscall"
)

func main() {
    // Opening a file triggers an open syscall
    file, err := os.Open("ch05-02-os-basics.md")
    if err != nil {
        fmt.Println("Could not open file:", err)
        return
    }
    defer file.Close()

    // Reading from a file triggers a read syscall
    buf := make([]byte, 100)
    n, err := file.Read(buf)
    if err != nil {
        fmt.Println("Could not read file:", err)
        return
    }
    fmt.Printf("Read %d bytes\n", n)

    // You can also make syscalls directly using syscall package
    // This gets the process ID using the getpid syscall
    pid := syscall.Getpid()
    fmt.Printf("My process ID: %d\n", pid)

    // Get the current working directory (uses getcwd syscall)
    cwd, err := os.Getwd()
    if err != nil {
        fmt.Println("Could not get cwd:", err)
        return
    }
    fmt.Printf("Current directory: %s\n", cwd)
}
```

On Linux, you can trace all syscalls a Go program makes using the `strace` tool:

```bash
strace -c ./my-program
```

The output shows you every syscall, how many times it was called, and how much time was spent in each one. It is really eye-opening to see how many syscalls happen even in a simple Go program.

## Why Go Developers Need to Know This

1. **Every I/O operation goes through the OS** - file reads, network calls, even printing to the terminal. Understanding syscalls helps you understand performance.

2. **Blocking syscalls and goroutines** - When a goroutine makes a blocking syscall (like waiting for network data), the Go runtime handles it so other goroutines can keep running. This is a big part of why Go is efficient.

3. **File descriptors are limited** - The OS limits how many files and sockets a process can open. In Go, this matters when building high-concurrency servers.

4. **Memory is virtual** - Go's garbage collector and memory allocator work with virtual memory provided by the OS. Understanding this helps you debug memory issues.

5. **Permissions and security** - The OS controls what your program can and cannot do. File permissions, network access, and user privileges all come from the OS.

The OS is not just a background thing. It is the foundation your Go code runs on, and understanding it makes you a much more effective developer.
