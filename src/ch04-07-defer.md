# Defer in Go

## What Is Defer?

The `defer` keyword schedules a function call to run when the **surrounding function returns**. It does not run immediately. Instead, Go pushes the deferred call onto a stack and pops them off in reverse order when the function finishes.

Think of defer as saying: "Do not forget to do this before you leave." It is Go's built-in way to handle cleanup tasks like closing files, releasing locks, and closing connections.

```go
func main() {
    fmt.Println("start")
    defer fmt.Println("deferred")
    fmt.Println("end")
}

// Output:
// start
// end
// deferred
```

See? "deferred" runs last, even though it was written in the middle. The `defer` statement does not execute immediately. It waits until `main()` is about to return.

## Common Uses

### Closing Files

This is the most classic use of defer. You open a file, and you want to make sure it gets closed no matter what happens:

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // guaranteed to close when readFile returns

    // do stuff with the file...
    // even if there is a panic, f.Close() will run
    return nil
}
```

Without defer, you would need to add `f.Close()` before every `return` statement. With defer, you write it once at the top and it always runs. This eliminates a whole class of resource leak bugs.

### Unlocking Mutexes

```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    defer mu.Unlock() // unlock when the function returns

    counter++
    // do more work...
    // Unlock is guaranteed, even if the code above panics
}
```

If you forget to unlock a mutex, other goroutines will wait forever (deadlock). Defer ensures the unlock always happens.

### Closing Database Connections

```go
func getUser(db *sql.DB, id int) (*User, error) {
    rows, err := db.Query("SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        return nil, err
    }
    defer rows.Close() // always close the rows

    // process rows...
    return user, nil
}
```

### Handling HTTP Response Bodies

```go
func fetchURL(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close() // always close the body

    return io.ReadAll(resp.Body)
}
```

## Multiple Defers Execute in LIFO Order

When you defer multiple calls, they execute in **Last In, First Out (LIFO)** order. The last deferred call runs first:

```go
func main() {
    defer fmt.Println("first")
    defer fmt.Println("second")
    defer fmt.Println("third")
    fmt.Println("done")
}

// Output:
// done
// third
// second
// first
```

Think of it like a stack. Each `defer` pushes onto the stack, and when the function returns, they pop off in reverse order.

This LIFO behavior makes sense for cleanup. If you open resource A, then open resource B, you should close B before closing A. Defer naturally handles this:

```go
func process() {
    f, _ := os.Open("file.txt")
    defer f.Close()       // closed second (LIFO)

    mu.Lock()
    defer mu.Unlock()     // unlocked first (LIFO)

    // do work...
}
```

## Defer with Arguments

Arguments to a deferred function are **evaluated at the time of the defer statement**, not at the time of execution. This is a subtle but important point:

```go
func main() {
    x := 10
    defer fmt.Println(x) // x is evaluated NOW, value is 10
    x = 20
    fmt.Println("end")
}

// Output:
// end
// 10    (not 20!)
```

Even though `x` changed to 20 after the defer, the deferred call prints 10. That is because the argument was evaluated when `defer` was called, not when the deferred function actually ran.

If you want the deferred function to see the current value at execution time, use a closure:

```go
func main() {
    x := 10
    defer func() {
        fmt.Println(x) // x is evaluated WHEN THIS RUNS
    }()
    x = 20
    fmt.Println("end")
}

// Output:
// end
// 20    (current value at execution time)
```

The key difference: `defer fmt.Println(x)` evaluates `x` immediately. `defer func() { fmt.Println(x) }()` creates a closure that captures `x` by reference, so it reads the value when it runs.

## Defer in Loops

Be careful with defer inside loops. Each iteration adds a deferred call, but none of them execute until the function returns. This can cause resource leaks:

```go
// BAD: all files stay open until the function returns
func processFiles(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close() // NOT closed until processFiles returns!
        // process f...
    }
    return nil
}
```

If there are 1000 files, all 1000 stay open at the same time. The fix is to move the loop body into a separate function:

```go
// GOOD: each file is closed after processing
func processFiles(paths []string) error {
    for _, path := range paths {
        if err := processOneFile(path); err != nil {
            return err
        }
    }
    return nil
}

func processOneFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // closed when processOneFile returns

    // process f...
    return nil
}
```

Now each file is closed right after it is processed, not at the end of the outer function.

## Performance Consideration

Defer has a small performance cost. For most code, this does not matter at all. But in extremely hot loops or performance-critical paths, you might want to call cleanup manually instead of using defer:

```go
// Fine for 99.9% of code
defer mu.Unlock()

// For the 0.1% hot path, manual unlock might be slightly faster
mu.Unlock()
```

In practice, I always use defer unless profiling shows it is a bottleneck. The correctness benefit far outweighs the tiny performance cost.

## The Defer / Panic / Recover Pattern

Go does not have exceptions, but you can use `panic` and `recover` with defer to catch panics:

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic occurred: %v", r)
        }
    }()

    return a / b, nil // panics if b is 0
}

func main() {
    result, err := safeDivide(10, 0)
    if err != nil {
        fmt.Println("Error:", err) // Error: panic occurred: integer divide by zero
    } else {
        fmt.Println("Result:", result)
    }
}
```

Here is how it works:

1. `defer` sets up a function with `recover()` before the risky code runs.
2. If the code panics, the deferred function runs.
3. `recover()` catches the panic value and stops the panic from crashing the program.
4. The function can return an error instead of crashing.

This pattern is useful in library code where you want to be resilient against panics in user-provided callbacks or risky operations.

## A Personal Note

Defer is one of those features I did not appreciate at first. "Why not just call Close() at the end?" I thought. Then I started writing functions with multiple return paths, error handling, and early returns. Suddenly, remembering to close every resource on every path became error-prone and annoying.

Defer solves this elegantly. You put the cleanup right next to the acquisition, and Go guarantees it runs. No matter how many return statements you add later, no matter what errors happen, the cleanup always executes.

Now I use defer instinctively. Open a file? Defer close. Lock a mutex? Defer unlock. Start a transaction? Defer rollback (or commit). It is a small keyword that makes resource management reliable, and I wish every language had it.
