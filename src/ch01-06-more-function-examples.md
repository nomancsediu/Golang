# More Function Examples in Go

## Practice Makes Progress

Reading about functions is one thing. Writing them is another. In this chapter, I want to share several practical function examples that I wrote while practicing. Each one teaches something slightly different, and together they build a solid understanding of how functions work in Go.

## Example 1: Calculator with Error Handling

This one uses multiple return values and the error pattern we just learned:

```go
package main

import (
    "errors"
    "fmt"
)

func calculate(a, b float64, operation string) (float64, error) {
    switch operation {
    case "+":
        return a + b, nil
    case "-":
        return a - b, nil
    case "*":
        return a * b, nil
    case "/":
        if b == 0 {
            return 0, errors.New("cannot divide by zero")
        }
        return a / b, nil
    default:
        return 0, errors.New("unknown operation: " + operation)
    }
}

func main() {
    ops := []struct {
        a, b      float64
        operation string
    }{
        {10, 5, "+"},
        {10, 5, "-"},
        {10, 5, "*"},
        {10, 5, "/"},
        {10, 0, "/"},
        {10, 5, "%"},
    }

    for _, op := range ops {
        result, err := calculate(op.a, op.b, op.operation)
        if err != nil {
            fmt.Printf("%.0f %s %.0f = Error: %s\n", op.a, op.operation, op.b, err)
        } else {
            fmt.Printf("%.0f %s %.0f = %.2f\n", op.a, op.operation, op.b, result)
        }
    }
}
```

What I learned here: the `switch` inside a function works great for handling different operations, and the error pattern makes it clear when something goes wrong.

## Example 2: Greeting Function

A simple function that generates a personalized greeting:

```go
package main

import "fmt"

func greet(name, timeOfDay string) string {
    var greeting string

    switch timeOfDay {
    case "morning":
        greeting = "Good morning"
    case "afternoon":
        greeting = "Good afternoon"
    case "evening":
        greeting = "Good evening"
    default:
        greeting = "Hello"
    }

    return greeting + ", " + name + "!"
}

func main() {
    fmt.Println(greet("Noman", "morning"))   // Good morning, Noman!
    fmt.Println(greet("Alice", "afternoon")) // Good afternoon, Alice!
    fmt.Println(greet("Bob", "evening"))     // Good evening, Bob!
    fmt.Println(greet("Eve", "night"))       // Hello, Eve!
}
```

This function takes two strings, builds a result using switch, and returns one string. Simple but practical.

## Example 3: Even or Odd Checker

```go
package main

import "fmt"

func isEven(n int) bool {
    return n%2 == 0
}

func main() {
    numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    for _, num := range numbers {
        if isEven(num) {
            fmt.Printf("%d is even\n", num)
        } else {
            fmt.Printf("%d is odd\n", num)
        }
    }
}
```

The `%` operator gives the remainder. If `n % 2` equals 0, the number is even. The function returns a `bool`, which we can use directly in an `if` condition.

## Example 4: Max of Two Numbers

```go
package main

import "fmt"

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func main() {
    fmt.Println(max(10, 20)) // 20
    fmt.Println(max(50, 30)) // 50
    fmt.Println(max(7, 7))   // 7
}
```

Wait, you might be thinking: does Go not have a built-in max function? Actually, Go 1.21 added a built-in `max` function! But writing it yourself is still great practice for understanding how functions work.

## Example 5: String Reverser

```go
package main

import "fmt"

func reverse(s string) string {
    // Convert string to a slice of runes to handle characters properly
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func main() {
    fmt.Println(reverse("hello"))     // olleh
    fmt.Println(reverse("Go is fun")) // nuf si oG
    fmt.Println(reverse("racecar"))   // racecar (palindrome!)
}
```

This one taught me something important: Go strings are UTF-8 encoded, and if you work with them byte by byte, non-ASCII characters can break. Using `[]rune` converts the string into individual Unicode characters, so this works correctly even with special characters.

The `for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1` part is a two-variable loop that swaps characters from both ends, moving toward the center.

## Example 6: Variadic Functions

A **variadic function** accepts any number of arguments of the same type. This is done with the `...` syntax:

```go
package main

import "fmt"

func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

func main() {
    fmt.Println(sum(1))            // 1
    fmt.Println(sum(1, 2))         // 3
    fmt.Println(sum(1, 2, 3))      // 6
    fmt.Println(sum(1, 2, 3, 4))   // 10
    fmt.Println(sum(1, 2, 3, 4, 5)) // 15

    // You can also pass a slice
    numbers := []int{10, 20, 30, 40}
    fmt.Println(sum(numbers...))   // 100
}
```

Inside the function, `nums` behaves like a **slice** of `int`. You can iterate over it, check its length, and do everything you would do with a normal slice.

Notice the `numbers...` syntax when passing a slice. The `...` after the slice name "unpacks" the slice into individual arguments.

## Example 7: Variadic with Other Parameters

You can mix regular parameters with variadic parameters, but the variadic one must be last:

```go
package main

import "fmt"

func formatMessage(prefix string, messages ...string) string {
    result := prefix + ": "
    for i, msg := range messages {
        if i > 0 {
            result += ", "
        }
        result += msg
    }
    return result
}

func main() {
    fmt.Println(formatMessage("ERROR"))                    // ERROR: 
    fmt.Println(formatMessage("INFO", "Server started"))   // INFO: Server started
    fmt.Println(formatMessage("DEBUG", "x=1", "y=2", "z=3")) // DEBUG: x=1, y=2, z=3
}
```

The `prefix` parameter is a regular string parameter. The `messages` parameter collects all remaining string arguments into a slice.

## Example 8: Fibonacci with Helper

This example uses a helper function to compute Fibonacci numbers:

```go
package main

import "fmt"

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func printFibonacci(count int) {
    fmt.Printf("First %d Fibonacci numbers: ", count)
    for i := 0; i < count; i++ {
        fmt.Printf("%d", fibonacci(i))
        if i < count-1 {
            fmt.Print(", ")
        }
    }
    fmt.Println()
}

func main() {
    printFibonacci(10)
}
```

Output:

```
First 10 Fibonacci numbers: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

Note: This recursive implementation is simple but very slow for large numbers. For production code, you would use an iterative approach or memoization. But for learning, recursion is a great concept to practice.

## What These Examples Taught Me

Writing these functions helped me understand several things:

**Functions should be small and focused.** Each of these examples does one clear thing. The `isEven` function only checks evenness. The `reverse` function only reverses. This makes them easy to understand, test, and reuse.

**Error handling becomes natural.** The calculator example shows how returning an error alongside the result makes the code robust without being complicated.

**Variadic functions are powerful.** Being able to accept any number of arguments is really handy for utility functions like `sum` or `formatMessage`.

**Helper functions improve readability.** The Fibonacci example uses two functions: one that computes a single number, and one that prints a sequence. Each is simple on its own.

The more I practice writing functions, the more natural it feels. It is like learning any skill: repetition builds intuition.
