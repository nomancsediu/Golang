# Function Return Values & Types in Go

## One of Go's Coolest Features

When I first saw a Go function return two values, I was confused. In most languages I had used before, a function returns one thing. Maybe an object, maybe an array, but conceptually it is one thing. Go just lets you return multiple values directly, and it feels so natural once you get used to it.

Let me walk through everything I have learned about return values in Go.

## Single Return Value

The simplest case is a function that returns one value:

```go
func square(n int) int {
    return n * n
}

func main() {
    result := square(5)
    fmt.Println(result) // 25
}
```

The `int` after the parameter list tells Go this function returns an integer. You must include a `return` statement that matches the declared type.

If you declare a return type but forget to return, Go gives you a compiler error:

```go
func broken(n int) int {
    fmt.Println(n)
    // missing return at end of function
}
```

This is helpful. The compiler catches the mistake before you ever run the code.

## Multiple Return Values

This is where Go gets interesting. A function can return more than one value:

```go
func divide(a, b float64) (float64, string) {
    if b == 0 {
        return 0, "cannot divide by zero"
    }
    return a / b, "success"
}

func main() {
    result, msg := divide(10, 3)
    fmt.Println(result) // 3.3333333333333335
    fmt.Println(msg)    // success

    result2, msg2 := divide(10, 0)
    fmt.Println(result2) // 0
    fmt.Println(msg2)    // cannot divide by zero
}
```

The return types are listed in parentheses: `(float64, string)`. When you call the function, you receive both values using the `:=` operator with two variables on the left side.

This is incredibly useful, and here is why: **the most common pattern in Go is returning a value and an error together.**

## The Value and Error Pattern

In Go, errors are not thrown like exceptions. They are returned as values. This means almost every function that could fail returns two things: the result and an error:

```go
func getUserAge(name string) (int, error) {
    if name == "" {
        return 0, fmt.Errorf("name cannot be empty")
    }
    // pretend we looked up the age in a database
    return 25, nil
}

func main() {
    age, err := getUserAge("Noman")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Age:", age) // Age: 25
}
```

The `nil` value means "no error." If `err` is `nil`, everything went fine. If `err` is not `nil`, something went wrong and you should handle it.

**This pattern is everywhere in Go.** Standard library functions use it. Third-party packages use it. It is the Go way of handling errors, and once you get used to it, it is very elegant.

## My Aha Moment About Multiple Returns

Coming from languages with try-catch exception handling, I was skeptical about returning errors. "Is this not tedious?" I thought. "Do I really need to check err after every function call?"

But then I realized something: **with exceptions, you never know what a function might throw.** You have to read the docs or hope for the best. With Go's approach, the function signature tells you exactly what could go wrong. If it returns an error, you know you need to handle it. The compiler even warns you if you ignore the error.

```go
// This will not compile if you do not use the err value
result, err := someFunction()
// If you write:
result, _ := someFunction()
// The _ discards the error, but at least you made a conscious choice
```

The `_` (underscore) is the **blank identifier**. It lets you discard a return value you do not care about. Use it sparingly with errors though, because ignoring errors can lead to silent bugs.

## Named Return Values

Go lets you name your return values. This can make your code more readable:

```go
func rectangleProps(length, width float64) (area, perimeter float64) {
    area = length * width
    perimeter = 2 * (length + width)
    return // naked return
}

func main() {
    a, p := rectangleProps(5, 3)
    fmt.Println("Area:", a)      // 15
    fmt.Println("Perimeter:", p) // 16
}
```

Notice the return types have names: `(area, perimeter float64)`. This does two things:

1. It documents what each return value means
2. It allows a **naked return** (just `return` without specifying values)

## Naked Returns

A **naked return** is when you write `return` without any values. Go automatically returns the named return variables:

```go
func swap(a, b int) (first, second int) {
    first = b
    second = a
    return // naked return, returns first and second
}
```

**My advice:** Use naked returns only in short, simple functions. In longer functions, they can be confusing because you have to remember what the named return variables are and what their current values are. Explicit returns are almost always clearer.

```go
// Better: explicit return
func swap(a, b int) (int, int) {
    return b, a
}
```

## Returning Different Types

Sometimes you need a function that can return different types. Go does not support this directly (no union types like TypeScript), but you can use an empty interface:

```go
func getValue(key string) interface{} {
    if key == "name" {
        return "Noman"
    }
    if key == "age" {
        return 25
    }
    return nil
}
```

This works, but it is not idiomatic Go. Usually, you should design your functions to return a specific type, or return a struct that contains the different possibilities. I am still learning the best patterns for this.

## A Practical Example: Safe Division

Here is a realistic example that uses multiple return values with error handling:

```go
package main

import (
    "errors"
    "fmt"
)

func safeDivide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero is not allowed")
    }
    return a / b, nil
}

func main() {
    results := []struct {
        a, b float64
    }{
        {10, 2},
        {15, 0},
        {7, 3},
    }

    for _, r := range results {
        result, err := safeDivide(r.a, r.b)
        if err != nil {
            fmt.Printf("%.1f / %.1f = Error: %s\n", r.a, r.b, err)
        } else {
            fmt.Printf("%.1f / %.1f = %.2f\n", r.a, r.b, result)
        }
    }
}
```

Output:

```
10.0 / 2.0 = 5.00
15.0 / 0.0 = Error: division by zero is not allowed
7.0 / 3.0 = 2.33
```

## Return Types Cheat Sheet

| Pattern | Syntax | Example |
|---------|--------|---------|
| Single return | `func f() int` | `return 42` |
| Multiple return | `func f() (int, string)` | `return 42, "ok"` |
| Named return | `func f() (n int, msg string)` | `return` (naked) or `return n, msg` |
| With error | `func f() (int, error)` | `return 0, err` or `return val, nil` |

## Key Takeaways

**Multiple return values** are one of Go's best features. They make error handling explicit and natural. Instead of throwing exceptions and hoping someone catches them, you return the error and the caller decides what to do.

**Named returns** can improve readability in short functions, but use naked returns sparingly.

**The value-error pattern** (`val, err := doSomething()`) is the standard Go way. You will see it everywhere, and you should use it in your own functions too.

**Never ignore errors** without a good reason. If a function returns an error, check it. Your future self will thank you.

Now let us practice with more function examples to really solidify these concepts.
