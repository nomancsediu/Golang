# Local Scope and Block Scope

## What Is Local Scope?

When you declare a variable inside a function, that variable has **local scope**. It only exists inside that function. No other function can see it or use it.

```go
package main

import "fmt"

func calculate() {
    result := 42 // local to calculate
    fmt.Println("Result:", result)
}

func main() {
    calculate()
    // fmt.Println(result) // ERROR: undefined: result
}
```

The variable `result` is local to `calculate()`. It is born when `calculate()` starts running, and it dies when `calculate()` returns. The `main()` function has no idea `result` ever existed.

This is a good thing. It means each function has its own private workspace. You do not have to worry about another function accidentally changing your variables.

## Block Scope Inside Control Structures

Go takes scope a step further with **block scope** inside control structures like `if`, `for`, and `switch`. Variables declared inside these blocks only exist within that block.

### Block Scope in `if` Statements

```go
package main

import "fmt"

func main() {
    score := 85

    if score >= 90 {
        grade := "A"
        fmt.Println("Grade:", grade) // works
    } else if score >= 80 {
        grade := "B"
        fmt.Println("Grade:", grade) // works
    } else {
        grade := "C"
        fmt.Println("Grade:", grade) // works
    }

    // fmt.Println(grade) // ERROR: undefined: grade
}
```

Each `grade` variable is separate. They all have the same name, but they live in different blocks. This is perfectly valid Go code, though it is a bit wasteful to declare the same variable three times.

A better approach would be to declare `grade` in the outer scope:

```go
package main

import "fmt"

func main() {
    score := 85
    var grade string

    if score >= 90 {
        grade = "A"
    } else if score >= 80 {
        grade = "B"
    } else {
        grade = "C"
    }

    fmt.Println("Grade:", grade) // works now
}
```

### Block Scope in `for` Loops

Variables declared inside a `for` loop are recreated on each iteration.

```go
package main

import "fmt"

func main() {
    for i := 0; i < 3; i++ {
        count := i * 2
        fmt.Printf("i=%d, count=%d\n", i, count)
    }

    // fmt.Println(i)     // ERROR: undefined: i
    // fmt.Println(count) // ERROR: undefined: count
}
```

The variable `i` is scoped to the `for` loop. The variable `count` is scoped to the loop body. Neither exists outside the loop.

### Block Scope in `switch` Statements

```go
package main

import "fmt"

func main() {
    day := "Monday"

    switch day {
    case "Monday":
        message := "Start of the week"
        fmt.Println(message)
    case "Friday":
        message := "Almost weekend"
        fmt.Println(message)
    default:
        message := "Regular day"
        fmt.Println(message)
    }

    // fmt.Println(message) // ERROR: undefined: message
}
```

Each `case` is its own block. Variables declared in one case do not leak into other cases or outside the switch.

## Variables Die When the Block Ends

This is one of the most important things to understand. When a block ends, all variables declared inside that block are destroyed. The memory they used can be reclaimed.

```go
package main

import "fmt"

func main() {
    {
        x := 100
        fmt.Println("Inside block, x =", x)
    }
    // x is gone now. The memory can be reused.

    {
        y := 200
        fmt.Println("Inside another block, y =", y)
    }
    // y is gone now too.
}
```

This is not just a rule. It is how Go manages memory efficiently. When variables go out of scope, the memory becomes available again.

## Accessing Outer Scope from Inner Block

A key rule: code inside an inner block can access variables from the outer scope. But the outer scope cannot access variables from the inner scope.

```go
package main

import "fmt"

func main() {
    greeting := "Hello" // outer scope

    {
        name := "Go learner" // inner scope
        // we can access both here
        fmt.Println(greeting, name) // works fine
    }

    fmt.Println(greeting) // works fine
    // fmt.Println(name) // ERROR: undefined: name
}
```

Think of it like looking out a window. From inside a room, you can see outside. But from outside, you cannot see into the room.

## The Most Common Mistake

I made this mistake many times when I started Go: declaring a variable inside an `if` block and then trying to use it outside.

```go
package main

import "fmt"

func main() {
    enabled := true

    if enabled {
        config := "production"
        fmt.Println("Config set to:", config)
    }

    // BIG MISTAKE: trying to use config outside the if block
    // fmt.Println(config) // ERROR: undefined: config
}
```

The fix is to declare the variable in the outer scope before the `if`:

```go
package main

import "fmt"

func main() {
    enabled := true
    var config string

    if enabled {
        config = "production"
    }

    fmt.Println("Config:", config) // works now
}
```

## Key Takeaways

- **Local scope**: variables inside a function are local to that function
- **Block scope**: variables inside `if`, `for`, `switch` only exist within that block
- Variables die when their block ends
- Inner blocks can access outer scope, but not the other way around
- The most common beginner mistake is trying to use a block variable outside its block
- Declare variables in the outer scope if you need them after the block ends

Understanding local and block scope is one of those things that seems obvious once you get it, but causes a lot of confusion when you are starting out. Trust me, I have been there.

---

*Next: [Package Scope](ch02-03-package-scope.md)*
