# Data Segment

## What Is the Data Segment?

The **data segment** is the part of memory where **global variables and static data** are stored. These are variables that exist for the entire lifetime of your program, from start to finish.

In Go, any variable declared at the package level (outside any function) lives in the data segment.

```go
package main

import "fmt"

// These variables live in the data segment
var appName = "GoLearner"
var maxConnections = 100
var isDebug = false

func main() {
    fmt.Println("App:", appName)
    fmt.Println("Max connections:", maxConnections)
    fmt.Println("Debug mode:", isDebug)
}
```

The variables `appName`, `maxConnections`, and `isDebug` are created when the program starts and destroyed when the program ends. They live in the data segment the entire time.

## Two Parts of the Data Segment

The data segment is divided into two parts:

### 1. Initialized Data Section

This section stores variables that have an **initial value** when the program starts. If you declare a variable with a value, it goes here.

```go
package main

import "fmt"

// Initialized data: these have values right away
var serverName = "production"
var port = 8080
var version = "3.2.1"
var enabled = true

func main() {
    fmt.Println(serverName, port, version, enabled)
}
```

### 2. Uninitialized Data Section (BSS)

**BSS** stands for "Block Started by Symbol." This section stores variables that are declared but **not given an initial value**. Go automatically initializes them to their **zero value**.

```go
package main

import "fmt"

// BSS section: no initial value, Go gives them zero values
var requestCount int    // zero value: 0
var lastUser string     // zero value: ""
var isReady bool        // zero value: false
var scores []int        // zero value: nil

func main() {
    fmt.Println("Request count:", requestCount) // 0
    fmt.Println("Last user:", lastUser)         // "" (empty string)
    fmt.Println("Is ready:", isReady)           // false
    fmt.Println("Scores:", scores)              // []
}
```

Even though we did not assign values, Go initializes them. Numbers become `0`, strings become `""`, booleans become `false`, and slices/maps/channels become `nil`.

## Package-Level Variables Live Here

Any variable declared outside a function is a package-level variable. These all live in the data segment:

```go
package main

import "fmt"

// All of these live in the data segment
var counter int = 0          // initialized data
var buffer []byte            // BSS (nil)
var config = map[string]string{ // initialized data
    "host": "localhost",
    "port": "3000",
}

// Constants also live in the data segment
const MaxRetries = 3
const DefaultTimeout = 30

func incrementCounter() {
    counter++ // modifying data segment variable
}

func main() {
    fmt.Println("Counter:", counter)
    incrementCounter()
    incrementCounter()
    fmt.Println("Counter after increments:", counter)
    fmt.Println("Config:", config)
    fmt.Println("Max retries:", MaxRetries)
}
```

Notice how `incrementCounter()` can modify `counter` because it is in the data segment and accessible from any function in the package.

## The Data Segment Exists for the Entire Program Lifetime

Unlike stack variables that come and go as functions are called and return, data segment variables exist from the moment the program starts until the moment it ends.

```go
package main

import "fmt"

var callCount = 0 // data segment: lives forever

func recordCall() {
    callCount++ // this value persists across calls
    fmt.Printf("Call #%d\n", callCount)
}

func main() {
    recordCall() // Call #1
    recordCall() // Call #2
    recordCall() // Call #3

    // callCount still has its value because it lives in data segment
    fmt.Println("Total calls:", callCount) // 3
}
```

Each time `recordCall()` is called, `callCount` still has its previous value. This is because `callCount` lives in the data segment, not on the stack.

## Why the Data Segment Matters

### 1. Global State

Data segment variables are a form of **global state**. Any function in the package can read and modify them. This is powerful but also dangerous. Too much global state makes your code hard to test and debug.

### 2. Memory Usage

Data segment variables are never freed. They occupy memory for the entire program lifetime. If you have many large package-level variables, they add up.

```go
package main

import "fmt"

// This 1MB buffer lives in data segment for the entire program
var bigBuffer = make([]byte, 1024*1024)

func main() {
    fmt.Println("Buffer size:", len(bigBuffer))
}
```

That 1MB buffer is never freed. It stays in memory until the program ends.

### 3. Goroutine Safety

If multiple goroutines access data segment variables, you need synchronization. Otherwise, you get **data races**.

```go
package main

import (
    "fmt"
    "sync"
)

var counter = 0
var mu sync.Mutex

func safeIncrement() {
    mu.Lock()
    counter++
    mu.Unlock()
}

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            safeIncrement()
            wg.Done()
        }()
    }

    wg.Wait()
    fmt.Println("Final counter:", counter) // 1000
}
```

Without the mutex, the counter would be unreliable because multiple goroutines would read and write it simultaneously.

### 4. Initialization Order

The order in which data segment variables are initialized matters. Go initializes them in declaration order, and the `init()` function runs after all package-level variables are initialized.

```go
package main

import "fmt"

var a = 10
var b = a * 2     // depends on a, which is already initialized
var c = b + a     // depends on a and b, both already initialized

func init() {
    fmt.Println("Init: a=%d, b=%d, c=%d", a, b, c)
}

func main() {
    fmt.Println(a, b, c) // 10, 20, 30
}
```

## Data Segment vs Stack vs Heap

Here is a quick comparison:

| Feature | Data Segment | Stack | Heap |
|---|---|---|---|
| What lives here | Global/static variables | Local variables | Dynamic allocations |
| Lifetime | Entire program | Function call | Until GC collects |
| Speed | Fast access | Very fast | Slower (needs GC) |
| Size | Fixed at compile time | Grows/shrinks | Grows as needed |
| Thread safety | Needs synchronization | Per-goroutine | Managed by GC |

## Key Takeaways

- The **data segment** stores package-level variables and constants
- It has two parts: **initialized data** and **uninitialized data (BSS)**
- Variables in the data segment exist for the **entire program lifetime**
- Go initializes BSS variables to their **zero values**
- Data segment variables are **global state** and need synchronization for goroutine safety
- Be mindful of memory usage: large package-level variables are never freed
- Understanding the data segment helps you understand global state and initialization

I used to declare all my variables at the package level because it was convenient. After learning about the data segment, I started being more careful. Now I only use package-level variables when I truly need global state, and I keep them small and synchronized.

---

*Next: [Stack in Go](ch02-09-stack.md)*
