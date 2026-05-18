# The Init Function in Go

## What Is init()?

Go has a special function called `init()`. It is special because it **runs automatically** before the `main()` function. You do not call it yourself. Go calls it for you.

```go
package main

import "fmt"

func init() {
    fmt.Println("This runs first!")
}

func main() {
    fmt.Println("This runs second!")
}
```

When you run this program, the output is:

```
This runs first!
This runs second!
```

The `init()` function ran before `main()` even though `main()` was where the program "starts." That is the magic of `init()`.

## Why Does init() Exist?

The `init()` function is designed for **setup and initialization**. It gives you a place to run code that needs to happen before your program starts doing its main work.

Common use cases include:

- **Loading configuration files**
- **Validating required environment variables**
- **Setting up database connections**
- **Registering handlers or plugins**
- **Initializing global variables that need computation**

## Multiple init() Functions in One File

You can have **multiple** `init()` functions in the same file. They execute in the **order they appear** in the file:

```go
package main

import "fmt"

func init() {
    fmt.Println("Init 1: Loading config")
}

func init() {
    fmt.Println("Init 2: Connecting to database")
}

func init() {
    fmt.Println("Init 3: Starting cache")
}

func main() {
    fmt.Println("Main: Application running")
}
```

Output:

```
Init 1: Loading config
Init 2: Connecting to database
Init 3: Starting cache
Main: Application running
```

Each `init()` runs in the order it is declared. This is predictable and easy to understand.

## init() in Multiple Files

When you have `init()` functions in different files within the same package, Go executes them based on the **alphabetical order of the file names**.

Imagine three files:

**a_setup.go:**
```go
package main

import "fmt"

func init() {
    fmt.Println("Init from a_setup.go")
}
```

**b_setup.go:**
```go
package main

import "fmt"

func init() {
    fmt.Println("Init from b_setup.go")
}
```

**c_setup.go:**
```go
package main

import "fmt"

func init() {
    fmt.Println("Init from c_setup.go")
}
```

The execution order would be:

```
Init from a_setup.go
Init from b_setup.go
Init from c_setup.go
```

Because `a_setup.go` comes before `b_setup.go` alphabetically. This is important to know if your init functions have dependencies between them.

## init() Runs Before main()

This is always true. No matter where you put `init()` in your file, it always runs before `main()`. Even if you put `init()` after `main()`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Main function")
}

func init() {
    fmt.Println("Init function (declared after main)")
}
```

Output:

```
Init function (declared after main)
Main function
```

The declaration order of `init()` vs `main()` in the file does not matter. `init()` always runs first.

## Real-World Example: Configuration Validation

Here is a practical example of using `init()` to validate configuration before the program starts:

```go
package main

import (
    "fmt"
    "os"
)

var dbURL string

func init() {
    // Load database URL from environment
    dbURL = os.Getenv("DATABASE_URL")
    if dbURL == "" {
        dbURL = "localhost:5432"
        fmt.Println("Warning: DATABASE_URL not set, using default")
    }
}

func init() {
    // Validate the URL is not empty
    if dbURL == "" {
        panic("Database URL cannot be empty")
    }
    fmt.Println("Database configuration validated")
}

func main() {
    fmt.Println("Connecting to database at:", dbURL)
}
```

The first `init()` loads the config. The second `init()` validates it. By the time `main()` runs, we know the configuration is valid.

## Real-World Example: Plugin Registration

A common pattern in Go is using `init()` to register plugins or handlers:

```go
package main

import "fmt"

// Registry to store handlers
var handlers = map[string]func(){}

func register(name string, handler func()) {
    handlers[name] = handler
}

func init() {
    register("greet", func() {
        fmt.Println("Hello!")
    })
}

func init() {
    register("farewell", func() {
        fmt.Println("Goodbye!")
    })
}

func main() {
    for name, handler := range handlers {
        fmt.Printf("Running handler: %s\n", name)
        handler()
    }
}
```

This pattern is used heavily in Go's standard library. For example, the `image` package uses `init()` to register image decoders.

## Things to Remember About init()

1. **init() takes no arguments** and returns nothing
2. **You cannot call init() yourself**; Go calls it automatically
3. **Multiple init() functions** are allowed and execute in order
4. **init() in multiple files** executes in alphabetical file order
5. **init() always runs before main()**
6. **Keep init() simple**; do not put complex logic in it
7. **Avoid dependencies between init() functions** in different files; the alphabetical order is fragile

## When Not to Use init()

While `init()` is useful, it can be overused. Avoid using it when:

- The initialization can fail and needs error handling (use explicit setup instead)
- The initialization is slow and might block startup
- You need control over when the setup happens
- The setup depends on runtime values that are not available at init time

For these cases, a regular setup function called from `main()` is better.

## Key Takeaways

- **init()** is a special function that runs automatically before `main()`
- You can have **multiple init()** functions; they run in declaration order
- **init() in different files** runs in alphabetical file name order
- Good for: **config loading, validation, registration, global setup**
- Keep init() simple and avoid complex logic
- For error-prone setup, prefer explicit function calls from `main()`

I used to put all my setup code in `main()`. Once I learned about `init()`, I started using it for simple, predictable initialization. But I also learned the hard way that putting too much logic in `init()` makes debugging harder, because you cannot control when it runs or handle errors gracefully.

---

*Next: [Internal Memory in Go](ch02-06-internal-memory.md)*
