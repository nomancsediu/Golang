# Ch03 09 Closure

A **closure** is a function that captures variables from its outer scope. Even after the outer function finishes running, the captured variables stay alive. This is one of the most important and most tricky concepts in Go.

## What Is a Closure?

When a function references a variable that is defined outside its own body, it "closes over" that variable. The function and the variables it captures together form a closure.

Here is the simplest example:

```go
package main

import "fmt"

func main() {
    message := "Hello"

    say := func() {
        fmt.Println(message) // captures message from outer scope
    }

    say() // Hello
}
```

The anonymous function `say` captures `message` from the `main` function. That is a closure. Simple enough.

But closures get more interesting when the outer function returns the inner function:

```go
package main

import "fmt"

func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

func main() {
    counter := makeCounter()
    fmt.Println(counter()) // 1
    fmt.Println(counter()) // 2
    fmt.Println(counter()) // 3
}
```

Here is what happens: `makeCounter` creates a variable `count` and returns an anonymous function. The returned function captures `count`. Even though `makeCounter` has finished running, `count` is still alive because the closure keeps a reference to it.

Each call to `counter()` increments and returns the same `count` variable. The closure "remembers" the state.

## Independent Closures

If you call `makeCounter` twice, you get two independent closures, each with its own `count`:

```go
func main() {
    c1 := makeCounter()
    c2 := makeCounter()

    fmt.Println(c1()) // 1
    fmt.Println(c1()) // 2
    fmt.Println(c2()) // 1 (independent counter)
    fmt.Println(c1()) // 3
    fmt.Println(c2()) // 2
}
```

`c1` and `c2` each have their own `count`. They do not share state.

## The Accumulator Pattern

Another common closure pattern is the accumulator:

```go
package main

import "fmt"

func makeAccumulator(start int) func(int) int {
    sum := start
    return func(n int) int {
        sum += n
        return sum
    }
}

func main() {
    acc := makeAccumulator(100)
    fmt.Println(acc(10)) // 110
    fmt.Println(acc(20)) // 130
    fmt.Println(acc(5))  // 135
}
```

The accumulator starts at 100 and adds each value you pass in. The `sum` variable is captured and persists across calls.

## Closures in Middleware

Closures are heavily used in Go middleware. A middleware function often captures configuration or state:

```go
package main

import (
    "fmt"
    "net/http"
)

func withRateLimit(maxRequests int) http.HandlerFunc {
    count := 0 // captured by the closure
    return func(w http.ResponseWriter, r *http.Request) {
        count++
        if count > maxRequests {
            http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        fmt.Fprintf(w, "Request #%d\n", count)
    }
}

func main() {
    http.HandleFunc("/", withRateLimit(5))
    http.ListenAndServe(":8080", nil)
}
```

The `count` variable is captured by the returned handler. Each request increments it. When it exceeds the limit, requests are rejected.

## The Classic Goroutine Bug

This is the bug that every Go developer makes at least once. It is the reason closures are considered tricky:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println(i) // BUG: all goroutines share the same i
        }()
    }
    time.Sleep(time.Second)
}
```

You might expect this to print 0, 1, 2, 3, 4. Instead, you will probably see something like:

```
5
5
5
5
5
```

Why? Because all the goroutines share the same variable `i`. By the time the goroutines start running, the loop has already finished and `i` is 5. Each goroutine's closure captures the same `i`, not a copy of its value at the time the goroutine was created.

### The Fix: Pass as a Parameter

The traditional fix is to pass `i` as a parameter to the goroutine function:

```go
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n) // each goroutine gets its own copy
    }(i)
}
```

Now each goroutine gets its own copy of `i` at the time it was created. The output will be 0, 1, 2, 3, 4 (in some order).

### Go 1.22+ Fixes This

Starting with Go 1.22, the loop variable is scoped to each iteration instead of the entire loop. This means the bug no longer occurs in the original code:

```go
// Go 1.22+: this now works correctly
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // each iteration gets its own i
    }()
}
```

If you are using Go 1.22 or later, the classic bug is fixed by the language itself. But it is still important to understand the concept, because you will see the old pattern in existing codebases.

## Closures and Mutable State

Closures can modify the variables they capture. This can be useful, but it can also lead to bugs if you are not careful:

```go
package main

import "fmt"

func main() {
    name := "Alice"

    changeName := func(newName string) {
        name = newName // modifies the captured variable
    }

    fmt.Println(name)        // Alice
    changeName("Bob")
    fmt.Println(name)        // Bob
}
```

The closure modifies `name` directly. There is no return value needed. The change happens through the captured reference.

## Closures in Functional Patterns

Closures enable functional programming patterns in Go:

```go
package main

import "fmt"

func makeAdder(x int) func(int) int {
    return func(y int) int {
        return x + y
    }
}

func main() {
    add5 := makeAdder(5)
    add10 := makeAdder(10)

    fmt.Println(add5(3))  // 8
    fmt.Println(add10(3)) // 13
}
```

`add5` and `add10` are both closures. They each capture a different value of `x` and use it when called.

## Personal Story About a Goroutine Bug

I once spent three hours debugging a web scraper. I was launching goroutines in a loop to fetch different URLs. Each goroutine was supposed to process a specific URL from a slice. But all the goroutines were processing the same URL, the last one in the slice. I printed the URLs inside the goroutines and they were all the same. That is when I realized I was capturing the loop variable.

The fix was one line: adding the URL as a parameter to the goroutine function instead of capturing it from the loop. Three hours of debugging for a one-line fix. That is the closure bug. Every Go developer has a story like this. Now I always pass loop variables as parameters to goroutines, even with Go 1.22. It is a good habit.

Closures are powerful, but they demand respect. Understand what your closures are capturing, and you will avoid the most common bugs in Go concurrent programming.
