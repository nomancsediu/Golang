# Ch03 04 IIFE

**IIFE** stands for **Immediately Invoked Function Expression**. That is a mouthful. Let me break it down. An IIFE is a function that runs as soon as you define it. You do not call it separately. It executes right there, right then.

## What Is an IIFE?

The idea is simple. You define a function, and you call it in the same expression. No separate call. No storing it in a variable first. Define and run, all at once.

```go
func() {
    fmt.Println("This runs immediately!")
}()
```

Look at the `()` at the very end. That is what calls the function. Without those parentheses, you just have a function that does nothing. With them, the function runs right away.

## IIFE with Parameters

You can pass arguments to an IIFE by adding values inside the final parentheses:

```go
func(name string) {
    fmt.Println("Hello,", name)
}("Gopher")
```

The `("Gopher")` at the end passes the argument to the function. The parameter `name` receives it, and the body runs immediately.

Here is an example with multiple parameters:

```go
package main

import "fmt"

func main() {
    func(a, b int) {
        sum := a + b
        fmt.Printf("%d + %d = %d\n", a, b, sum)
    }(10, 20)
}
```

Output:

```
10 + 20 = 30
```

## Why Use IIFE?

You might wonder why you would ever use an IIFE instead of just writing the code directly. There are a few good reasons.

### 1. Limit Variable Scope

Variables declared inside an IIFE do not leak out. They are scoped to the function. This keeps your namespace clean.

```go
package main

import "fmt"

func main() {
    // Without IIFE, x would be available in the entire main function
    func() {
        x := 42
        fmt.Println("Inside IIFE:", x)
    }()

    // x is not accessible here
    // fmt.Println(x) // compile error: undefined: x

    fmt.Println("Outside IIFE")
}
```

If you have a block of code that needs temporary variables and you do not want those variables floating around in the outer scope, wrap the code in an IIFE.

### 2. Initialization Logic

Sometimes you need to run some setup code and compute a value. An IIFE can do both in one expression:

```go
package main

import "fmt"

func main() {
    config := func() map[string]string {
        cfg := make(map[string]string)
        cfg["host"] = "localhost"
        cfg["port"] = "8080"
        cfg["mode"] = "development"
        return cfg
    }()

    fmt.Println("Config:", config)
}
```

The IIFE creates and returns a map, and `config` gets the result. The temporary variable `cfg` is scoped to the IIFE. Clean and contained.

### 3. Create Isolated Scope in Tests

IIFEs are useful in tests where you want each test case to have its own scope:

```go
func TestCalculations(t *testing.T) {
    // Each IIFE is its own scope
    func() {
        result := add(2, 3)
        if result != 5 {
            t.Errorf("expected 5, got %d", result)
        }
    }()

    func() {
        result := add(10, -3)
        if result != 7 {
            t.Errorf("expected 7, got %d", result)
        }
    }()
}
```

### 4. IIFEs with Goroutines

A very common pattern is starting a goroutine with an IIFE that captures values:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for i := 0; i < 3; i++ {
        go func(n int) {
            fmt.Printf("Goroutine %d\n", n)
        }(i)
    }

    time.Sleep(time.Second)
}
```

By passing `i` as a parameter to the IIFE, each goroutine gets its own copy of the value. This avoids the classic closure-over-loop-variable bug.

## IIFE That Returns a Value

An IIFE can also return a value. You can use the return value directly in an assignment:

```go
package main

import "fmt"

func main() {
    max := func(a, b int) int {
        if a > b {
            return a
        }
        return b
    }(17, 42)

    fmt.Println("Max is:", max) // Max is: 42
}
```

The IIFE compares two numbers and returns the larger one. The result is assigned to `max` immediately.

## Personal Note

I used to think IIFEs were just a JavaScript thing. When I saw them in Go, I was surprised. But they make a lot of sense. The biggest benefit I have found is keeping variable scope tight. When I have a chunk of code that needs temporary variables, I wrap it in an IIFE instead of polluting the outer scope. It keeps things clean.

The goroutine pattern with IIFEs is especially important. If you are starting goroutines in a loop, passing the loop variable as a parameter to the IIFE is the safe way to do it. We will talk more about this in the closure chapter, but for now, just know that IIFEs and goroutines are best friends.
