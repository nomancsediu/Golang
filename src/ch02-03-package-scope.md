# Package Scope

## What Is Package Scope?

When you declare a variable **outside of any function**, that variable has **package scope**. It is accessible from anywhere within the same package.

This is different from local scope. Local variables only exist inside their function. Package-level variables exist for the entire lifetime of the program and can be used by any function in the package.

```go
package main

import "fmt"

// package-level variable: accessible anywhere in this package
var appName = "MyGoApp"

func printApp() {
    // we can access appName here
    fmt.Println("App:", appName)
}

func main() {
    // we can also access appName here
    fmt.Println("Welcome to", appName)
    printApp()
}
```

Both `main()` and `printApp()` can use `appName` because it is declared at the package level.

## Exported vs Unexported

Go has a simple but powerful rule for package scope:

- **Uppercase first letter** = **exported** (public, accessible from other packages)
- **Lowercase first letter** = **unexported** (private, only accessible within the same package)

This applies to variables, functions, types, constants, and struct fields.

### Exported Example

Create a file called `config/config.go`:

```go
package config

// exported: other packages can access this
var Version = "2.1.0"

// exported: other packages can call this
func GetInfo() string {
    return "Config v" + Version
}

// unexported: only accessible within package config
var secretKey = "hidden123"

// unexported: only accessible within package config
func validate() bool {
    return secretKey != ""
}
```

Then in your main program:

```go
package main

import (
    "fmt"
    "myproject/config"
)

func main() {
    // exported: we can access this
    fmt.Println(config.Version)
    fmt.Println(config.GetInfo())

    // UNEXPORTED: these will cause compile errors
    // fmt.Println(config.secretKey) // ERROR
    // config.validate()            // ERROR
}
```

The variables and functions starting with uppercase letters are accessible from outside the package. The ones starting with lowercase letters are not.

## Package Scope and Initialization Order

When you have multiple package-level variables, Go initializes them in order. It follows the order they appear in the source file, from top to bottom.

```go
package main

import "fmt"

var a = 10
var b = a + 5
var c = b * 2

func main() {
    fmt.Println(a) // 10
    fmt.Println(b) // 15
    fmt.Println(c) // 30
}
```

Go initializes `a` first, then `b` (which depends on `a`), then `c` (which depends on `b`). The order matters.

### Be Careful with Circular Dependencies

You cannot have package-level variables that depend on each other in a circular way:

```go
package main

// This will cause a compilation error
var x = y + 1
var y = x + 1 // ERROR: initialization loop
```

Go detects this cycle and refuses to compile. The order must be clear and non-circular.

## Package Scope Across Multiple Files

If a package has multiple `.go` files, all package-level variables are shared across those files. They all belong to the same package.

Imagine two files in the same directory:

**file1.go:**
```go
package main

var greeting = "Hello from file1"
```

**file2.go:**
```go
package main

import "fmt"

func main() {
    fmt.Println(greeting) // works! same package
}
```

Even though `greeting` is declared in `file1.go`, `file2.go` can access it because they are in the same package.

## Constants at Package Level

Constants follow the same scope rules as variables. You can declare package-level constants:

```go
package main

import "fmt"

// package-level constant
const MaxRetries = 3
const DefaultTimeout = 30

func retry() {
    for i := 0; i < MaxRetries; i++ {
        fmt.Println("Attempt", i+1)
    }
}

func main() {
    fmt.Println("Max retries:", MaxRetries)
    fmt.Println("Timeout:", DefaultTimeout)
    retry()
}
```

Using constants for fixed values is a good practice. It makes your code clearer and prevents accidental modification.

## When to Use Package-Level Variables

Package-level variables are useful, but you should use them carefully. Here are good use cases:

- **Configuration values** that the whole program needs
- **Constants** for magic numbers and strings
- **Logger instances** shared across the package
- **Database connections** that multiple functions use

Avoid using package-level variables for things that should be local:

- Temporary calculation results
- Loop counters
- Function-specific data

Too many package-level variables make your code harder to understand and test. They create hidden connections between functions.

## Key Takeaways

- **Package scope** means variables declared outside any function are accessible anywhere in the package
- **Uppercase** first letter = exported (public), **lowercase** = unexported (private)
- Package-level variables are initialized in **declaration order**
- All files in the same package share package-level variables
- Use package-level variables sparingly and intentionally
- Constants at package level are a great practice for shared values

When I first learned about exported vs unexported, I was confused by the uppercase/lowercase rule. It felt unusual compared to `public`/`private` keywords in other languages. But now I love it. It is simple, it is visible right in the code, and there are no extra keywords to remember.

---

*Next: [Variable Shadowing](ch02-04-variable-shadowing.md)*
