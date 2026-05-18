# Ch03 08 Higher Order Function

A **higher-order function** is a function that takes another function as a parameter, or returns a function, or both. This is where functions go from being simple tools to being building blocks for architecture.

## What Is a Higher-Order Function?

There are two rules. If a function satisfies either one, it is a higher-order function:

1. It takes a **function as a parameter**
2. It **returns a function**

```go
// Higher-order: takes a function as parameter
func apply(n int, fn func(int) int) int {
    return fn(n)
}

// Higher-order: returns a function
func multiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}
```

Both of these are higher-order functions. The first one takes a function. The second one returns one. Either way, functions are flowing in or out.

## Why Higher-Order Functions Are Powerful

Higher-order functions give you **flexibility**. Instead of hardcoding behavior, you let the caller decide what happens. This means you write less code and reuse more.

Think about it this way: a first-order function is a tool that does one specific job. A higher-order function is a tool that can be configured to do many jobs. It is the difference between a hammer and a Swiss Army knife.

## Example: Custom Filter Function

```go
package main

import "fmt"

func filter(nums []int, test func(int) bool) []int {
    var result []int
    for _, n := range nums {
        if test(n) {
            result = append(result, n)
        }
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    evens := filter(numbers, func(n int) bool {
        return n%2 == 0
    })
    fmt.Println("Evens:", evens) // [2 4 6 8 10]

    odds := filter(numbers, func(n int) bool {
        return n%2 != 0
    })
    fmt.Println("Odds:", odds) // [1 3 5 7 9]

    big := filter(numbers, func(n int) bool {
        return n > 5
    })
    fmt.Println("Big:", big) // [6 7 8 9 10]
}
```

One `filter` function, three different behaviors. That is the power of higher-order functions.

## Example: Custom Map Function

```go
package main

import "fmt"

func mapInts(nums []int, transform func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = transform(n)
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    doubled := mapInts(numbers, func(n int) int {
        return n * 2
    })
    fmt.Println("Doubled:", doubled) // [2 4 6 8 10]

    squared := mapInts(numbers, func(n int) int {
        return n * n
    })
    fmt.Println("Squared:", squared) // [1 4 9 16 25]

    negated := mapInts(numbers, func(n int) int {
        return -n
    })
    fmt.Println("Negated:", negated) // [-1 -2 -3 -4 -5]
}
```

## Example: Function That Returns a Function

This is sometimes called a **factory function**. You call it with some configuration, and it returns a customized function:

```go
package main

import "fmt"

func makeGreeter(greeting string) func(string) string {
    return func(name string) string {
        return greeting + ", " + name + "!"
    }
}

func main() {
    hello := makeGreeter("Hello")
    hi := makeGreeter("Hi")
    howdy := makeGreeter("Howdy")

    fmt.Println(hello("Alice"))  // Hello, Alice!
    fmt.Println(hi("Bob"))       // Hi, Bob!
    fmt.Println(howdy("Carol"))  // Howdy, Carol!
}
```

The `makeGreeter` function creates a custom greeting function. Each one "remembers" the greeting it was created with. This is also a closure, which we will cover next.

## Real-World Examples from the Standard Library

Go's standard library uses higher-order functions a lot:

**sort.Slice** takes a comparison function:

```go
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
```

**strings.Map** takes a mapping function:

```go
upper := strings.Map(func(r rune) rune {
    return unicode.ToUpper(r)
}, "hello")
```

**filepath.Walk** takes a visitor function:

```go
filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
    fmt.Println(path)
    return nil
})
```

## The Middleware Pattern

The middleware pattern is one of the most important uses of higher-order functions in Go web development. A **middleware** is a function that takes a handler function and returns a wrapped handler function:

```go
package main

import (
    "fmt"
    "net/http"
)

// Middleware: takes a handler, returns a handler
func withLogging(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("Request: %s %s\n", r.Method, r.URL.Path)
        next(w, r)
    }
}

func home(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Welcome home!")
}

func main() {
    // Wrap home handler with logging middleware
    http.HandleFunc("/", withLogging(home))
    http.ListenAndServe(":8080", nil)
}
```

The `withLogging` function is a higher-order function. It takes `home` and returns a new function that logs the request and then calls `home`. This pattern lets you stack middleware:

```go
handler := withAuth(withLogging(withCORS(home)))
```

Each middleware wraps the next one. Clean, composable, and powerful.

## Personal Note

Higher-order functions changed how I think about code structure. Before I understood them, I would write the same filtering or mapping logic over and over with slight variations. Now I write the pattern once and pass the varying logic as a function. The middleware pattern in particular is something I use in every Go web project. It keeps the code clean and lets me add cross-cutting concerns like logging, authentication, and CORS without touching the core logic. Higher-order functions are not just a language feature. They are a design tool.
