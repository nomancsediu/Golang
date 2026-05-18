# Stack in Go

## What Is the Stack?

The **stack** is a region of memory used for **function execution**. It works like a stack of plates: you can only add or remove items from the top. This is called **LIFO** (Last In, First Out).

When a function is called, Go pushes a **stack frame** onto the stack. When the function returns, Go pops the frame off. This is how Go keeps track of function calls and local variables.

```go
package main

import "fmt"

func first() {
    fmt.Println("In first()")
}

func second() {
    first()
    fmt.Println("In second()")
}

func main() {
    second()
    fmt.Println("In main()")
}
```

The call stack looks like this during execution:

```
When main() calls second():
  Stack: [main]

When second() calls first():
  Stack: [main, second, first]

When first() returns:
  Stack: [main, second]

When second() returns:
  Stack: [main]

When main() returns:
  Stack: []
```

Each function gets its own space on the stack, and when it is done, that space is freed.

## Each Goroutine Gets Its Own Stack

In Go, every **goroutine** gets its own stack. This is important. It means goroutines do not share stack memory, which makes concurrent programming safer.

When you start a goroutine with `go func()`, Go creates a new stack for it. The main goroutine also has its own stack.

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int) {
    // This variable lives on this goroutine's stack
    taskName := fmt.Sprintf("Task-%d", id)
    fmt.Println("Worker", id, "starting:", taskName)
    time.Sleep(100 * time.Millisecond)
    fmt.Println("Worker", id, "done")
}

func main() {
    // Each goroutine has its own stack
    for i := 1; i <= 3; i++ {
        go worker(i)
    }

    time.Sleep(500 * time.Millisecond)
    fmt.Println("All done")
}
```

Each `worker` goroutine has its own `taskName` variable on its own stack. They do not interfere with each other.

## Stack Frames

A **stack frame** is the block of memory on the stack that belongs to a single function call. It contains:

1. **Local variables** declared in the function
2. **Parameters** passed to the function
3. **Return address** (where to go back when the function finishes)
4. **Base pointer** of the calling function (to restore the previous frame)

```go
package main

import "fmt"

func add(a, b int) int {
    // Stack frame for add() contains:
    // - parameters: a, b
    // - local variable: result
    // - return address back to main()
    result := a + b
    return result
}

func main() {
    // Stack frame for main() contains:
    // - local variable: sum
    x := 3
    y := 5
    sum := add(x, y)
    fmt.Println("Sum:", sum)
}
```

When `add()` is called:
1. A new stack frame is pushed for `add()`
2. Parameters `a` and `b` are placed in the frame
3. Local variable `result` is placed in the frame
4. When `add()` returns, the frame is popped

## The Stack Is Fast

Stack allocation is **very fast**. There is no searching for free space, no garbage collection, no complex allocation logic. Pushing and popping stack frames is just moving a pointer.

This is one of the reasons Go functions are efficient. Local variables that stay on the stack have essentially zero allocation cost.

```go
package main

import "fmt"

func fastCalc(n int) int {
    // All these local variables go on the stack
    // Allocation is basically free (just moving a pointer)
    a := n * 2
    b := a + 10
    c := b / 3
    return c
}

func main() {
    for i := 0; i < 5; i++ {
        result := fastCalc(i)
        fmt.Printf("fastCalc(%d) = %d\n", i, result)
    }
}
```

Each call to `fastCalc()` gets a stack frame with `a`, `b`, and `c`. When the function returns, the frame is popped. No garbage collection needed.

## Stack Size in Go

Go stacks start **small** and **grow as needed**. Initially, each goroutine's stack is only about **2KB** (it used to be 4KB or 8KB in older Go versions, but it has been reduced).

If a function needs more stack space, Go automatically grows the stack. When the function returns and the stack is no longer needed, Go can shrink it back.

```go
package main

import "fmt"

// This recursive function will grow the stack
func recursive(n int) int {
    if n <= 0 {
        return 0
    }
    // Each recursive call pushes a new stack frame
    local := n * 2
    return local + recursive(n-1)
}

func main() {
    result := recursive(100)
    fmt.Println("Result:", result)
}
```

Go handles this stack growth automatically. You do not need to worry about it in most cases.

## Stack Overflow

If a function calls itself too many times (infinite recursion), the stack will keep growing until it runs out of space. This is a **stack overflow**.

```go
package main

import "fmt"

func infinite(n int) {
    // This will eventually cause a stack overflow
    fmt.Println("Call depth:", n)
    infinite(n + 1) // calls itself forever
}

func main() {
    infinite(1)
}
```

Running this will produce an error like:

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

Go sets a maximum stack size (default 1GB on 64-bit systems) to prevent a single goroutine from consuming all memory.

## Why Understanding the Stack Matters

### 1. Understanding Function Calls

When you understand stack frames, you understand what happens when functions call each other. Each call adds a frame, each return removes one.

### 2. Recursion

Recursive functions push a new frame for each recursive call. Too many recursive calls means too many frames, which means stack overflow. This is why deep recursion is dangerous.

### 3. Performance

Variables on the stack are fast. Variables that escape to the heap are slower (they need garbage collection). Understanding the stack helps you keep variables on the stack when possible.

### 4. Debugging

When a Go program crashes, you often see a **stack trace**. Understanding stack frames helps you read these traces.

```
goroutine 1 [running]:
main.crash()
    /home/user/main.go:8 +0x27
main.main()
    /home/user/main.go:13 +0x1f
```

This trace shows that `main()` called `crash()`, and `crash()` is where the problem occurred.

## Key Takeaways

- The **stack** is a LIFO data structure for function execution
- Each **goroutine** gets its own stack
- A **stack frame** contains local variables, parameters, and return address
- Stack allocation is **very fast** (just moving a pointer)
- Go stacks start at about **2KB** and grow/shrink as needed
- **Stack overflow** happens when the stack grows beyond its limit (usually from infinite recursion)
- Understanding the stack helps with debugging, performance, and understanding function calls

Learning about the stack was when Go really started making sense to me. I could finally read stack traces, understand why recursive functions could be dangerous, and appreciate why Go functions are so fast. The stack is one of the most fundamental concepts in programming, and it is worth understanding well.

---

*Next: [Heap in Go](ch02-10-heap.md)*
