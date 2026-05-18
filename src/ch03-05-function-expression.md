# Ch03 05 Function Expression

A **function expression** is when you assign an anonymous function to a variable. The variable then becomes a function that you can call. It is similar to named functions, but there are important differences that are worth understanding.

## The Basic Idea

Here is a function expression:

```go
add := func(a, b int) int {
    return a + b
}
```

You define an anonymous function and assign it to the variable `add`. Now `add` is a function. You can call it just like any other function:

```go
result := add(3, 4)
fmt.Println(result) // 7
```

The variable `add` holds a function value. It is not storing the result of the function. It is storing the function itself. When you call `add(3, 4)`, you are calling the anonymous function that was assigned to it.

## Function Expression vs Named Function

These two look similar, but they behave differently:

```go
// Named function
func add(a, b int) int {
    return a + b
}

// Function expression
add := func(a, b int) int {
    return a + b
}
```

The key differences:

**Named functions** can be referenced before they are declared. The Go compiler processes all named function declarations before executing the code.

**Function expressions** follow the normal rules of variable assignment. You must define them before you use them.

```go
package main

import "fmt"

func main() {
    // This works: named function is visible everywhere in the package
    fmt.Println(namedAdd(3, 4)) // 7

    // This would NOT work if declared below the call
    // fmt.Println(exprAdd(3, 4)) // compile error

    exprAdd := func(a, b int) int {
        return a + b
    }

    fmt.Println(exprAdd(3, 4)) // 7
}

func namedAdd(a, b int) int {
    return a + b
}
```

The `namedAdd` function can be called before its definition because Go processes named function declarations at the package level. The `exprAdd` variable must be declared before it is used, just like any other variable.

## Function Expressions Capture Variables (Closures)

This is where function expressions get interesting. When you define a function expression, it can capture variables from its surrounding scope. This is called a **closure**.

```go
package main

import "fmt"

func main() {
    multiplier := 3

    multiply := func(n int) int {
        return n * multiplier // captures multiplier from outer scope
    }

    fmt.Println(multiply(5))  // 15
    fmt.Println(multiply(10)) // 30

    // Changing the captured variable affects the function
    multiplier = 5
    fmt.Println(multiply(5))  // 25
}
```

Notice how changing `multiplier` changes the behavior of `multiply`. The function expression does not copy the value of `multiplier`. It captures a reference to it. When `multiplier` changes, the function sees the new value.

This is different from passing a parameter:

```go
package main

import "fmt"

func main() {
    multiplier := 3

    // Parameter: value is copied at call time
    multiply := func(n, m int) int {
        return n * m
    }

    fmt.Println(multiply(5, multiplier)) // 15

    multiplier = 5
    // The function still needs the new value passed explicitly
    fmt.Println(multiply(5, multiplier)) // 25
}
```

In the second example, `multiplier` is a parameter, not a captured variable. The function does not "remember" it. You have to pass it every time.

## Function Expressions as Struct Fields

You can store function expressions in structs:

```go
package main

import "fmt"

type Calculator struct {
    operation func(int, int) int
    name      string
}

func main() {
    calc := Calculator{
        name: "adder",
        operation: func(a, b int) int {
            return a + b
        },
    }

    fmt.Printf("%s: %d\n", calc.name, calc.operation(10, 20)) // adder: 30

    calc.operation = func(a, b int) int {
        return a * b
    }
    calc.name = "multiplier"

    fmt.Printf("%s: %d\n", calc.name, calc.operation(10, 20)) // multiplier: 200
}
```

## Function Expressions in Slices and Maps

Since function expressions are values, you can store them in collections:

```go
package main

import "fmt"

func main() {
    operations := map[string]func(int, int) int{
        "add": func(a, b int) int {
            return a + b
        },
        "sub": func(a, b int) int {
            return a - b
        },
        "mul": func(a, b int) int {
            return a * b
        },
    }

    fmt.Println(operations["add"](10, 5))  // 15
    fmt.Println(operations["sub"](10, 5))  // 5
    fmt.Println(operations["mul"](10, 5))  // 50
}
```

This pattern is useful for dispatch tables and command patterns.

## When to Use Function Expressions

**Use function expressions when:**
- You need to capture variables from the surrounding scope
- You want to store a function in a variable, slice, or map
- You are passing a function as an argument
- The function is only needed in a specific context

**Use named functions when:**
- The function is called from multiple places
- You need the function to be visible before its definition
- The function is complex and deserves a descriptive name
- The function is part of a package API

## Personal Note

Function expressions felt natural to me because I had used them in JavaScript. But the closure behavior in Go caught me off guard. When I first saw that changing a captured variable changes the function behavior, I had to think about it for a while. It makes sense once you understand that the function captures a reference, not a copy. Just be careful with closures in loops. That is a bug I have made more than once, and we will cover it in the closure chapter.
