# Variable Shadowing in Go

## What Is Variable Shadowing?

**Variable shadowing** happens when you declare a variable in an inner scope with the same name as a variable in an outer scope. The inner variable "shadows" the outer one. Within the inner scope, the name refers to the inner variable, not the outer one.

Go allows shadowing. It is not an error. But it can be very confusing and can cause bugs that are hard to spot.

```go
package main

import "fmt"

func main() {
    name := "Alice"

    {
        name := "Bob" // this shadows the outer "name"
        fmt.Println(name) // prints "Bob"
    }

    fmt.Println(name) // prints "Alice" (outer variable is unchanged)
}
```

Inside the inner block, `name` refers to "Bob". Outside the block, `name` still refers to "Alice". The inner `name` does not overwrite the outer one. It just hides it temporarily.

## Shadowing Is Not Reassignment

This is a critical distinction. Shadowing creates a **new variable**. Reassignment changes the **existing variable**.

```go
package main

import "fmt"

func main() {
    x := 10

    // This is SHADOWING: creates a new x
    {
        x := 20
        fmt.Println("Inner x:", x) // 20
    }
    fmt.Println("Outer x:", x) // 10 (unchanged)

    // This is REASSIGNMENT: changes the existing x
    x = 30
    fmt.Println("After reassignment:", x) // 30
}
```

The `:=` operator inside the block creates a new variable. The `=` operator outside reassigns the existing one. This difference is everything.

## Shadowing with if Statements

A very common place where shadowing happens is inside `if` statements:

```go
package main

import "fmt"

func main() {
    status := "unknown"

    ok := true
    if ok {
        status := "confirmed" // shadows outer status
        fmt.Println("Inside if:", status) // confirmed
    }

    fmt.Println("Outside if:", status) // unknown
}
```

If you wanted to change the outer `status`, you should use `=` instead of `:=`:

```go
package main

import "fmt"

func main() {
    status := "unknown"

    ok := true
    if ok {
        status = "confirmed" // reassigns outer status
    }

    fmt.Println("Status:", status) // confirmed
}
```

## Shadowing with for Loops

The `for` loop variable often shadows outer variables:

```go
package main

import "fmt"

func main() {
    i := 100

    for i := 0; i < 3; i++ { // this i shadows the outer i
        fmt.Println("Loop i:", i) // 0, 1, 2
    }

    fmt.Println("Outer i:", i) // 100 (unchanged)
}
```

Again, the loop creates its own `i`. The outer `i` is not affected.

## Shadowing with Function Parameters

Function parameters can also be shadowed inside the function body:

```go
package main

import "fmt"

func greet(name string) {
    fmt.Println("Hello,", name)

    // shadowing the parameter
    name := name + " (shadowed)" // creates a new variable
    fmt.Println("Shadowed:", name)
}

func main() {
    greet("Alice")
}
```

Wait, actually that code above would cause a compilation error because you cannot use `:=` to redeclare a variable in the same scope. Let me fix that:

```go
package main

import "fmt"

func greet(name string) {
    fmt.Println("Hello,", name)

    // shadowing inside an inner block
    {
        name := name + " (shadowed)"
        fmt.Println("Shadowed:", name)
    }

    fmt.Println("Original:", name)
}

func main() {
    greet("Alice")
}
```

Inside the inner block, `name` gets shadowed. Outside, the original parameter is still there.

## Why Shadowing Is Dangerous

Shadowing can cause bugs that are really hard to find. Here is why:

1. **You think you are modifying the outer variable, but you are not.** This is the most common problem.
2. **The code compiles fine.** Go does not warn you about shadowing by default.
3. **It is hard to spot when reading code.** The variable names look the same.

### My Shadowing Bug Story

I once spent two hours debugging a function that was supposed to update a counter. The code looked something like this:

```go
package main

import "fmt"

var count = 0

func increment() {
    // I thought I was updating the package-level count
    // but I was actually creating a new local variable!
    count := count + 1 // SHADOWING, not reassignment
    fmt.Println("Inside:", count)
}

func main() {
    increment()
    increment()
    increment()
    fmt.Println("Global count:", count) // still 0!
}
```

I expected the global `count` to be 3, but it was still 0. The `:=` inside `increment()` created a new local `count` that shadowed the package-level one. The fix was simple:

```go
count = count + 1 // reassignment, not shadowing
```

But finding that bug took me a long time, because the code looked correct at first glance.

## How to Detect Shadowing

Go does not warn about shadowing by default, but there are tools that can help:

### Using `go vet`

```bash
go vet ./...
```

`go vet` can catch some shadowing issues, but not all of them.

### Using `shadow` Tool

There is a specialized linter called `shadow` that detects variable shadowing:

```bash
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
shadow ./...
```

### Using golangci-lint

The `golangci-lint` tool includes shadow detection:

```bash
golangci-lint run --enable govet
```

I now run `golangci-lint` on all my projects. It catches shadowing and many other issues before they become bugs.

## Best Practices to Avoid Shadowing Bugs

1. **Use `=` instead of `:=` when you want to modify an existing variable**
2. **Use different variable names** when you need a separate variable
3. **Run a linter** to catch accidental shadowing
4. **Keep your scopes small** so there are fewer chances for shadowing
5. **Be extra careful** when working with package-level variables

## Key Takeaways

- **Shadowing** happens when an inner scope declares a variable with the same name as an outer scope
- Shadowing creates a **new variable**; it does not modify the outer one
- Go allows shadowing, but it can cause **confusing bugs**
- Use `=` for reassignment, `:=` for new variables
- Use **linting tools** to detect accidental shadowing
- I learned about shadowing the hard way: through a bug that took hours to find

Shadowing is one of those things that seems harmless until it bites you. Now that I know about it, I am much more careful with my variable names and declarations.

---

*Next: [The Init Function](ch02-05-init-function.md)*
