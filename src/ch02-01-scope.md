# Scope in Go

## What Is Scope?

**Scope** is the region of code where a variable can be accessed. That is it. Simple concept, but it has big implications.

When you declare a variable in Go, that variable is not accessible from everywhere in your program. It has a "home range," and outside that range, the variable does not exist.

Think of it like this: if you declare a variable inside a function, it is like a person living in a house. You can find them inside the house, but if you go to a different house, that person is not there.

## Go Is Block-Scoped

Go uses **block scoping**. This means that variables are scoped to the block they are declared in. A block is any code surrounded by curly braces `{}`.

Here are some examples of blocks:

- A function body is a block
- The body of an `if` statement is a block
- The body of a `for` loop is a block
- The body of a `switch` statement is a block
- Even a standalone pair of `{}` is a block

```go
package main

import "fmt"

func main() {
    // x is scoped to main function
    x := 10
    fmt.Println(x)

    {
        // inner block
        y := 20
        fmt.Println(y) // works, y is in this block
        fmt.Println(x) // works, x is from outer block
    }

    // fmt.Println(y) // ERROR: y is not defined here
}
```

In the code above, `y` only exists inside the inner block. Once that block ends, `y` is gone.

## Why Scope Matters

Scope is not just a rule the compiler enforces. It serves important purposes:

### 1. Prevent Bugs

When variables have limited scope, you cannot accidentally use them in the wrong place. The compiler catches the error early.

```go
package main

import "fmt"

func main() {
    if true {
        count := 5
    }
    // If we could use count here, it would be confusing
    // Is count 5? Is it 0? Is it even valid?
    // Scope rules prevent this confusion
    // fmt.Println(count) // ERROR: undefined: count
}
```

### 2. Avoid Naming Conflicts

You can reuse variable names in different scopes without conflict. This is very useful.

```go
package main

import "fmt"

func process() {
    name := "Alice"
    fmt.Println("Processing:", name)
}

func main() {
    name := "Bob"
    fmt.Println("Hello:", name)
    process()
}
```

Both functions have a variable called `name`, but they are completely separate. No conflict, no confusion.

### 3. Memory Management

When a variable goes out of scope, Go knows it can reclaim the memory. This is how the garbage collector knows what to clean up. Variables with smaller scope free memory sooner.

## Scope Levels in Go

Go has several levels of scope, from narrow to wide:

| Scope Level | Where Declared | Accessible From |
|---|---|---|
| **Block scope** | Inside `{}` within a function | Only within that block |
| **Function scope** | Inside a function | Only within that function |
| **Package scope** | Outside all functions | Anywhere in the same package |
| **Exported scope** | Outside all functions, uppercase name | Anywhere that imports the package |

## A Simple Example Showing Multiple Scopes

```go
package main

import "fmt"

// package scope: accessible anywhere in this package
var version = "1.0.0"

func main() {
    // function scope: accessible anywhere in main
    language := "Go"

    fmt.Println("Learning", language, "version", version)

    for i := 0; i < 3; i++ {
        // block scope: i only exists inside this for loop
        message := fmt.Sprintf("Attempt %d", i)
        fmt.Println(message)
    }

    // fmt.Println(i)       // ERROR: undefined: i
    // fmt.Println(message) // ERROR: undefined: message

    fmt.Println("Done with", language)
}
```

Notice how `i` and `message` only exist inside the `for` loop. Once the loop ends, they are gone.

## Key Takeaways

- **Scope** defines where a variable can be accessed
- Go is **block-scoped**: variables live within their `{}`
- Smaller scope is better: it prevents bugs and helps memory management
- Variables in inner blocks can access outer scope, but not the other way around
- Understanding scope prevents many common Go errors

When I first learned this, I kept making the mistake of trying to use loop variables outside the loop. The compiler errors were frustrating, but they were actually protecting me from bugs. Once I internalized the scope rules, my code got much cleaner.

---

*Next: [Local and Block Scope](ch02-02-local-block-scope.md)*
