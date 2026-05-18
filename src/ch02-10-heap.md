# Heap in Go

## What Is the Heap?

The **heap** is a region of memory used for **dynamic allocation**. Unlike the stack, which automatically manages memory through function calls, the heap is where data lives when:

1. The size is **not known at compile time**
2. The data needs to **survive after a function returns**
3. The data is **too large** for the stack

The heap is more flexible than the stack, but it comes with a cost: heap memory must be managed by the **garbage collector**, which adds overhead.

```go
package main

import "fmt"

func createGreeting(name string) *string {
    // This string escapes to the heap because we return a pointer to it
    greeting := "Hello, " + name
    return &greeting
}

func main() {
    msg := createGreeting("Go learner")
    fmt.Println(*msg) // the greeting still exists because it is on the heap
}
```

If `greeting` were on the stack, it would be destroyed when `createGreeting()` returns. But because we return a pointer to it, Go moves it to the heap so it survives.

## Escape Analysis

Go uses **escape analysis** to decide whether a variable goes on the stack or the heap. The compiler analyzes your code and determines if a variable "escapes" its function.

A variable **escapes** when:

- You return a pointer to it
- You store a reference to it in a variable that lives longer
- You send it to a channel
- You store it in a slice that grows
- You close over it in a goroutine

### Seeing Escape Analysis

You can see the compiler's escape analysis decisions using the `-gcflags` flag:

```bash
go build -gcflags="-m" main.go
```

Let me show you an example:

```go
package main

import "fmt"

func noEscape() int {
    // x stays on the stack
    // it does not escape
    x := 42
    return x
}

func doesEscape() *int {
    // y escapes to the heap
    // because we return a pointer
    y := 42
    return &y
}

func main() {
    a := noEscape()
    b := doesEscape()

    fmt.Println(a, *b)
}
```

Running `go build -gcflags="-m"` might show output like:

```
./main.go:7:6: can inline noEscape
./main.go:13:6: can inline doesEscape
./main.go:14:2: y escapes to heap
./main.go:18:13: inlining call to noEscape
./main.go:19:13: inlining call to doesEscape
```

The key line is: `y escapes to heap`. The compiler tells you that `y` could not stay on the stack because a pointer to it escapes the function.

## Common Escape Scenarios

### 1. Returning a Pointer

```go
package main

import "fmt"

func newValue() *int {
    x := 10
    return &x // x escapes to heap
}

func main() {
    ptr := newValue()
    fmt.Println(*ptr) // 10
}
```

The variable `x` must go to the heap because we return a pointer to it. If it were on the stack, it would be destroyed when `newValue()` returns.

### 2. Storing in a Slice That Grows

```go
package main

import "fmt"

func buildSlice() []int {
    // The backing array of this slice may escape to heap
    s := make([]int, 0)
    for i := 0; i < 100; i++ {
        s = append(s, i) // append may cause reallocation
    }
    return s
}

func main() {
    nums := buildSlice()
    fmt.Println("Length:", len(nums))
    fmt.Println("First:", nums[0])
}
```

Slices returned from functions almost always have their backing array on the heap.

### 3. Capturing in a Closure

```go
package main

import (
    "fmt"
    "time"
)

func startCounter() {
    // count escapes to heap because the goroutine closure captures it
    count := 0

    go func() {
        for i := 0; i < 5; i++ {
            count++
            fmt.Println("Count:", count)
            time.Sleep(100 * time.Millisecond)
        }
    }()
}

func main() {
    startCounter()
    time.Sleep(1 * time.Second)
}
```

The variable `count` must go to the heap because the goroutine closure captures it. The goroutine might outlive the function call, so `count` needs to survive.

### 4. Interface Values

```go
package main

import "fmt"

func getValue() interface{} {
    x := 42
    return x // x escapes to heap because it is stored in an interface
}

func main() {
    val := getValue()
    fmt.Println(val)
}
```

When you assign a value to an interface, it often escapes to the heap because the interface needs to hold a copy of the value.

## Heap Is Slower Than Stack

This is the key performance difference. Stack allocation is nearly free. Heap allocation requires:

1. **Finding free space** on the heap
2. **Allocating the memory**
3. **Garbage collecting** it later when it is no longer needed

The garbage collector adds overhead. It needs to scan memory, find unreachable objects, and free them. This takes CPU time.

```go
package main

import "fmt"

// Stack version: fast
func stackVersion(n int) int {
    total := 0
    for i := 0; i < n; i++ {
        temp := i * 2  // stays on stack
        total += temp
    }
    return total
}

// Heap version: slower
func heapVersion(n int) *int {
    total := 0
    for i := 0; i < n; i++ {
        temp := i * 2
        total += temp
    }
    return &total // escapes to heap
}

func main() {
    fmt.Println("Stack result:", stackVersion(1000))
    result := heapVersion(1000)
    fmt.Println("Heap result:", *result)
}
```

The `stackVersion` is faster because all variables stay on the stack. The `heapVersion` is slower because `total` escapes to the heap.

## How to Minimize Heap Allocations

Here are some tips to keep variables on the stack:

1. **Return values, not pointers**, when the data is small
2. **Pre-allocate slices** with `make()` when you know the size
3. **Use value types** instead of pointer types when possible
4. **Avoid unnecessary interface conversions**
5. **Check escape analysis** when performance matters

```go
package main

import "fmt"

// Good: return value (stays on stack)
func getPoint() (int, int) {
    x := 10
    y := 20
    return x, y // values, no escape
}

// Less good: return pointer (escapes to heap)
func getPointPtr() *[]int {
    p := []int{10, 20}
    return &p // escapes to heap
}

func main() {
    x, y := getPoint()
    fmt.Println(x, y)

    p := getPointPtr()
    fmt.Println(*p)
}
```

## Key Takeaways

- The **heap** is for dynamic allocation when data needs to survive function returns
- Go uses **escape analysis** to decide stack vs heap
- Use `go build -gcflags="-m"` to see escape analysis results
- Variables escape when you return pointers, store in growing slices, capture in goroutines, or use interfaces
- **Heap is slower** than stack because it needs garbage collection
- To minimize heap allocations: return values instead of pointers, pre-allocate slices, use value types
- Understanding escape analysis helps you write more performant Go code

When I first ran `go build -gcflags="-m"` on my code, I was surprised by how many variables were escaping to the heap. It motivated me to think more carefully about when I use pointers and when I use values. Not every escape is a problem, but knowing about them helps you make informed decisions.

---

*Next: [Garbage Collection](ch02-11-garbage-collection.md)*
