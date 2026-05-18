# Ch03 03 Anonymous Functions

An **anonymous function** is a function without a name. You define it inline, right where you need it. I see anonymous functions everywhere in Go code. Once you understand them, you start seeing them in almost every codebase.

## What Is an Anonymous Function?

A regular function has a name. An anonymous function does not. That is the only difference. Everything else works the same way. You define parameters, you write a body, and you return values. The syntax is almost identical, just without the name after `func`.

Here is a regular function:

```go
func add(a, b int) int {
    return a + b
}
```

Here is the same thing as an anonymous function:

```go
func(a, b int) int {
    return a + b
}
```

Notice there is no name. That is it. Same logic, no name.

## Syntax: Defining and Calling

You can define an anonymous function and call it immediately:

```go
result := func(a, b int) int {
    return a + b
}(3, 4)

fmt.Println(result) // 7
```

See the `(3, 4)` at the end? That is calling the function right after defining it. The whole expression defines the function and invokes it in one step.

You can also assign an anonymous function to a variable and call it later:

```go
add := func(a, b int) int {
    return a + b
}

fmt.Println(add(3, 4)) // 7
fmt.Println(add(10, 5)) // 15
```

Now `add` is a variable that holds a function. You can call it like any other function.

## Passing Anonymous Functions as Arguments

One of the most common uses of anonymous functions is passing them as arguments to other functions:

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    people := []string{"Charlie", "Alice", "Bob", "Diana"}

    sort.Slice(people, func(i, j int) bool {
        return people[i] < people[j]
    })

    fmt.Println(people) // [Alice Bob Charlie Diana]
}
```

The `sort.Slice` function takes a slice and a comparison function. We pass an anonymous function as the comparison. This is so common in Go that you will see this pattern in almost every codebase.

Here is another example with a custom function:

```go
package main

import "fmt"

func apply(nums []int, operation func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = operation(n)
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    doubled := apply(numbers, func(n int) int {
        return n * 2
    })

    fmt.Println(doubled) // [2 4 6 8 10]
}
```

## Using Anonymous Functions with Goroutines

This is probably the most common place you will see anonymous functions in Go. When you want to start a goroutine, you often use an anonymous function:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    message := "Hello from goroutine"

    go func() {
        fmt.Println(message)
    }()

    time.Sleep(time.Second)
    fmt.Println("Main function done")
}
```

The `go func() { ... }()` pattern starts a new goroutine with an anonymous function. You will see this pattern constantly in Go programs. It is the standard way to run code concurrently.

## Common Pattern with Closures

Anonymous functions often capture variables from their surrounding scope. This is called a **closure**, and we will cover it in detail later. For now, just know that anonymous functions can "see" variables around them:

```go
package main

import "fmt"

func main() {
    prefix := "Result: "

    calculate := func(a, b int) string {
        sum := a + b
        return prefix + fmt.Sprintf("%d", sum)
    }

    fmt.Println(calculate(3, 4))  // Result: 7
    fmt.Println(calculate(10, 5)) // Result: 15
}
```

The anonymous function captures `prefix` from the outer scope. This is what makes anonymous functions so flexible.

## Anonymous Functions vs Named Functions

When should you use anonymous functions vs named functions?

**Use named functions** when:
- The function is called from multiple places
- The function has complex logic that deserves a name
- You need to reference the function by name (like for recursion)

**Use anonymous functions** when:
- The function is used only once, right where it is defined
- The function is short and simple
- You need to capture variables from the surrounding scope
- You are passing it as an argument to another function

## Personal Note

When I first started learning Go, anonymous functions felt weird. Why would you write a function without a name? Then I started reading Go code on GitHub and realized that anonymous functions are everywhere. Goroutines use them. Sorting uses them. HTTP handlers use them. Middleware uses them. They are not weird. They are essential. Now I use them all the time without even thinking about it.

The trick is knowing when to use them and when not to. If an anonymous function gets longer than 5 or 6 lines, I usually extract it into a named function. Short and focused anonymous functions are great. Long and complex anonymous functions are hard to read and hard to test.
