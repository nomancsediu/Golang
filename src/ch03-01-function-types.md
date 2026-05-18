# Ch03 01 Function Types

When I first saw a **function type** in Go, I was confused. I understood `int`, `string`, and `bool` as types. But a function as a type? That felt weird. Once I got it though, everything clicked. Function types are one of the most useful features in Go.

## What Is a Function Type?

A **function type** is exactly what it sounds like. It is a type that describes the signature of a function. You define it using the `type` keyword followed by the `func` keyword and the function signature.

```go
type MyFunc func(int) int
```

This line creates a new type called `MyFunc`. Any function that takes an `int` and returns an `int` matches this type. That is all there is to it.

## Defining Function Types

You can define function types with any signature you need:

```go
// A function that takes two ints and returns an int
type BinaryOp func(int, int) int

// A function that takes a string and returns nothing
type StringHandler func(string)

// A function that takes nothing and returns a string and an error
type Factory func() (string, error)

// A function that takes an int and returns a function that returns an int
type IntGenerator func(int) func() int
```

The key thing to understand is that the **function type only describes the signature**. It does not contain any logic. The logic comes from the actual function you assign to it.

## Using Function Types as Variables

Once you define a function type, you can create variables of that type and assign functions to them:

```go
package main

import "fmt"

type MathOp func(int, int) int

func main() {
    var op MathOp

    op = func(a, b int) int {
        return a + b
    }
    fmt.Println(op(3, 4)) // 7

    op = func(a, b int) int {
        return a * b
    }
    fmt.Println(op(3, 4)) // 12
}
```

The variable `op` can hold any function that matches the `MathOp` signature. This is powerful because you can swap out the behavior without changing the code that calls it.

## Passing Functions as Arguments

This is where function types really shine. You can write functions that accept other functions as arguments:

```go
package main

import "fmt"

type Filter func(int) bool

func filterNumbers(nums []int, f Filter) []int {
    var result []int
    for _, v := range nums {
        if f(v) {
            result = append(result, v)
        }
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    isEven := func(n int) bool {
        return n%2 == 0
    }

    evens := filterNumbers(numbers, isEven)
    fmt.Println("Even numbers:", evens) // [2 4 6 8 10]

    isGreaterThan5 := func(n int) bool {
        return n > 5
    }

    big := filterNumbers(numbers, isGreaterThan5)
    fmt.Println("Greater than 5:", big) // [6 7 8 9 10]
}
```

The `filterNumbers` function does not care what the filter logic is. It just applies whatever function you pass in. That is flexibility.

## Returning Functions

You can also return functions from other functions:

```go
package main

import "fmt"

type Transformer func(int) int

func getTransformer(doubled bool) Transformer {
    if doubled {
        return func(n int) int {
            return n * 2
        }
    }
    return func(n int) int {
        return n + 1
    }
}

func main() {
    t1 := getTransformer(true)
    fmt.Println(t1(5)) // 10

    t2 := getTransformer(false)
    fmt.Println(t2(5)) // 6
}
```

## Function Type as a Struct Field

You can use function types as fields in structs. This is a common pattern for callbacks and strategies:

```go
package main

import "fmt"

type Processor struct {
    transform func(int) int
    name      string
}

func main() {
    p := Processor{
        name: "doubler",
        transform: func(n int) int {
            return n * 2
        },
    }

    fmt.Printf("%s: %d\n", p.name, p.transform(5)) // doubler: 10
}
```

This pattern is useful when you want an object to have configurable behavior without using interfaces.

## Why Function Types Matter

Function types are not just a cool feature. They are the backbone of several important patterns in Go:

**Callbacks** - Pass a function to be called later when something happens.

**Strategy Pattern** - Swap out the behavior of a function by passing different function implementations.

**Middleware Pattern** - A function that takes a handler function and returns a wrapped handler function. This is how most Go web frameworks implement middleware.

**Event Systems** - Store function types in maps or slices to be called when events fire.

## Personal Note

When I started reading Go standard library code, I saw function types everywhere. The `http.Handler` interface uses them. The `sort` package uses them. The `testing` package uses them. Once I understood function types, reading Go code became so much easier. If you are struggling with this concept, just keep practicing. Write a few examples yourself. It will click.

The next chapter covers named functions in more detail.
