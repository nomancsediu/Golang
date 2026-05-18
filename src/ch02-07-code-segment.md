# Code Segment

## What Is the Code Segment?

The **code segment** (also called the **text segment**) is the part of memory where your program's **compiled instructions** are stored. When you write Go code and compile it, the compiler translates your code into machine instructions. Those machine instructions live in the code segment.

Think of it this way: if your program is a recipe, the code segment is where the written recipe is kept. The CPU reads the instructions from the code segment and follows them step by step.

## The Code Segment Is Read-Only

The code segment is **read-only**. This means your program cannot modify it while running. The instructions are loaded once when the program starts, and they stay the same until the program ends.

This is an important security feature. If programs could modify their own instructions, it would open the door to all kinds of attacks. By making the code segment read-only, the operating system protects your program's instructions from being tampered with.

```go
package main

import "fmt"

// The compiled machine code for this function
// lives in the code segment
func greet(name string) string {
    return "Hello, " + name
}

// The compiled machine code for main() also
// lives in the code segment
func main() {
    message := greet("Go learner")
    fmt.Println(message)
}
```

When this program runs, the CPU fetches the instructions for `main()` from the code segment. When `main()` calls `greet()`, the CPU jumps to the address in the code segment where `greet()`'s instructions are stored.

## Functions Live in the Code Segment

Every function you define in your Go program gets compiled into machine code and placed in the code segment. This includes:

- Your own functions
- Methods on your types
- Functions from imported packages
- Functions from the standard library (linked at compile time)

```go
package main

import "fmt"

func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

func main() {
    // When main() runs, the CPU fetches instructions from code segment
    sum := add(2, 3)           // CPU jumps to add() in code segment
    product := multiply(2, 3)  // CPU jumps to multiply() in code segment
    fmt.Println(sum, product)  // CPU jumps to fmt.Println in code segment
}
```

Each function call is essentially the CPU jumping to a different address in the code segment.

## How Function Calls Work

When you call a function in Go, here is what happens at a low level:

1. The CPU is currently executing instructions at some address in the code segment
2. When it encounters a function call, it saves the current position (return address)
3. The CPU jumps to the address of the called function in the code segment
4. The function executes its instructions
5. When the function returns, the CPU jumps back to the saved return address

```go
package main

import "fmt"

func step1() {
    fmt.Println("Step 1: Prepare")
}

func step2() {
    fmt.Println("Step 2: Execute")
}

func step3() {
    fmt.Println("Step 3: Complete")
}

func main() {
    // CPU executes main() instructions
    step1() // CPU jumps to step1() in code segment, then returns
    step2() // CPU jumps to step2() in code segment, then returns
    step3() // CPU jumps to step3() in code segment, then returns
    fmt.Println("All steps done")
}
```

The CPU bounces around the code segment as functions call each other. The return addresses are saved on the stack so the CPU knows where to go back to.

## Why the Code Segment Matters

You might wonder why you need to know about the code segment. Here are some reasons:

### 1. Understanding Function Calls

When you understand that functions live in the code segment and calls are jumps between addresses, function calls make more sense. You understand why there is a small overhead for each call (the CPU needs to save state and jump).

### 2. Security

The read-only nature of the code segment prevents certain types of attacks. Understanding this helps you appreciate why buffer overflows are dangerous (they can overwrite return addresses on the stack, redirecting execution to malicious code).

### 3. Function Pointers

In Go, functions are first-class citizens. You can assign functions to variables and pass them as arguments. Under the hood, these are addresses in the code segment.

```go
package main

import "fmt"

func add(a, b int) int {
    return a + b
}

func subtract(a, b int) int {
    return a - b
}

func main() {
    // operation holds an address in the code segment
    var operation func(int, int) int

    operation = add
    fmt.Println("5 + 3 =", operation(5, 3)) // 8

    operation = subtract
    fmt.Println("5 - 3 =", operation(5, 3)) // 2
}
```

The variable `operation` holds the address of a function in the code segment. When you call `operation(5, 3)`, the CPU jumps to whatever address `operation` points to.

### 4. Shared Libraries

The code segment can be shared between multiple instances of the same program. If you run the same Go program 10 times, the operating system can share one copy of the code segment across all 10 processes. This saves memory.

## Code Segment Size

The code segment size depends on how much code your program has. A small Go program might have a code segment of a few kilobytes. A large application could have a code segment of many megabytes.

You can check the size of your Go binary, which includes the code segment:

```bash
go build -o myapp main.go
ls -lh myapp
```

Go binaries tend to be larger than equivalent C programs because Go includes the runtime (garbage collector, scheduler, etc.) in every binary.

## Key Takeaways

- The **code segment** stores your program's compiled machine instructions
- It is **read-only** for security and stability
- **Functions** live in the code segment; calling a function means the CPU jumps to that address
- Function calls have a small overhead because the CPU must save state and jump
- **Function variables** in Go hold addresses in the code segment
- The code segment can be **shared** between multiple instances of the same program
- Understanding the code segment helps you understand function calls, security, and performance

I used to think of functions as just "blocks of code." Learning that they are actual addresses in memory that the CPU jumps to was a small but meaningful shift in my understanding. It made the idea of function pointers and callbacks much less mysterious.

---

*Next: [Data Segment](ch02-08-data-segment.md)*
