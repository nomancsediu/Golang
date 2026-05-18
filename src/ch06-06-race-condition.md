# Race Conditions in Go

## What is a Race Condition?

A **race condition** happens when two or more goroutines access the same data at the same time, and at least one of them is writing, and there is no synchronization to control the access order. The result depends on the timing of the goroutines, which is unpredictable.

Think of it like two people editing the same document at the same time without knowing about each other. One person writes something, the other person overwrites it, and the final document is a mess. Neither person intended to cause problems, but the lack of coordination led to corrupted data.

Race conditions are one of the most dangerous bugs in concurrent programming because:

- **Results are unpredictable**. The program might work correctly 99 times out of 100 and fail on the 100th run.
- **They are hard to reproduce**. The bug depends on the exact timing of goroutine scheduling, which changes every run.
- **They are silent**. There is no crash, no error message. The data just gets corrupted silently.
- **They are hard to debug**. Adding print statements changes the timing, which can make the bug disappear.

## A Classic Example: The Broken Counter

Here is the most common race condition example. Two goroutines incrementing the same counter:

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    var counter int
    var wg sync.WaitGroup

    numGoroutines := 1000
    wg.Add(numGoroutines)

    for i := 0; i < numGoroutines; i++ {
        go func() {
            defer wg.Done()
            counter++ // RACE CONDITION! Multiple goroutines read/write counter
        }()
    }

    wg.Wait()
    fmt.Printf("Counter: %d (expected %d)\n", counter, numGoroutines)
}
```

If you run this, you might get 1000 on some runs, but more likely you will get something like 987 or 943 or 965. It changes every time. Why?

Because `counter++` is not one atomic operation. Under the hood, it is three steps:

1. **Read** the current value of counter (let us say it is 42)
2. **Add 1** to get 43
3. **Write** 43 back to counter

Now imagine two goroutines doing this at the same time:

```
Goroutine A: reads counter = 42
Goroutine B: reads counter = 42
Goroutine A: calculates 42 + 1 = 43
Goroutine B: calculates 42 + 1 = 43
Goroutine A: writes 43
Goroutine B: writes 43
```

Both goroutines read 42, both write 43. The counter should be 44, but it is 43. One increment was lost. This is called a **lost update**.

## The Race Detector: Your Best Friend

Go has a built-in **race detector** that catches these bugs. Just add the `-race` flag when you run or build your program:

```bash
go run -race main.go
```

Let us run it on the broken counter example:

```
==================
WARNING: DATA RACE
Write at 0x00c0000b2008 by goroutine 7:
  main.main.func1()
      /path/main.go:16 +0x72

Previous write at 0x00c0000b2008 by goroutine 6:
  main.main.func1()
      /path/main.go:16 +0x72
==================
Counter: 987 (expected 1000)
Found 1 data race(s)
```

The race detector tells you exactly which line of code has the race, which goroutines are involved, and whether it is a read or a write conflict. It is incredibly detailed and helpful.

**Always use the race detector during development and testing.** It catches bugs that you might never find manually.

## How to Fix Race Conditions

There are three main ways to fix a race condition in Go:

### 1. Use a Mutex

A **mutex** ensures only one goroutine can access the data at a time:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var counter int
    var wg sync.WaitGroup
    var mu sync.Mutex

    numGoroutines := 1000
    wg.Add(numGoroutines)

    for i := 0; i < numGoroutines; i++ {
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }

    wg.Wait()
    fmt.Printf("Counter: %d (expected %d)\n", counter, numGoroutines)
}
```

Now the output is always 1000. The mutex ensures that only one goroutine increments the counter at a time.

### 2. Use Atomic Operations

For simple counter operations, the **`sync/atomic`** package is even faster than a mutex:

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    var counter int64
    var wg sync.WaitGroup

    numGoroutines := 1000
    wg.Add(numGoroutines)

    for i := 0; i < numGoroutines; i++ {
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }

    wg.Wait()
    fmt.Printf("Counter: %d (expected %d)\n", counter, numGoroutines)
}
```

`atomic.AddInt64` performs the read-modify-write as a single indivisible operation. No goroutine can interfere.

### 3. Use a Channel

Use a channel to serialize access through a single goroutine that owns the data:

```go
package main

import "fmt"

func main() {
    counter := 0
    increments := make(chan int, 1000)
    done := make(chan bool)

    // Single goroutine owns the counter
    go func() {
        for inc := range increments {
            counter += inc
        }
        done <- true
    }()

    for i := 0; i < 1000; i++ {
        increments <- 1
    }
    close(increments)

    <-done
    fmt.Printf("Counter: %d (expected 1000)\n", counter)
}
```

This follows Go's philosophy: **share memory by communicating**. The counter is only ever accessed by one goroutine, so there is no race.

## Common Patterns That Cause Races

Here are some patterns I have learned to watch out for:

### Checking a Flag and Then Acting

```go
// RACE CONDITION!
var ready bool

go func() {
    // Do some setup
    ready = true
}()

if ready { // Another goroutine might change ready between check and use
    // Use the thing that should be ready
}
```

This is the classic "check-then-act" race. The solution is to use a channel or sync.Once to signal when setup is complete.

### Appending to a Slice

```go
// RACE CONDITION!
var results []int
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(val int) {
        defer wg.Done()
        results = append(results, val) // Multiple goroutines appending!
    }(i)
}
wg.Wait()
```

Multiple goroutines appending to the same slice is a race. Fix it with a mutex, a channel, or pre-allocate and write to specific indices.

### Reading and Writing a Map

```go
// RACE CONDITION!
var cache = make(map[string]string)

go func() {
    cache["key"] = "value" // Write
}()

go func() {
    _ = cache["key"] // Read while another goroutine writes
}()
```

Use `sync.RWMutex` to protect the map, or use `sync.Map` for concurrent map access.

## My Race Condition Story

I was working on a web service that counted API requests. We had a global counter that every request handler incremented. In development with low traffic, everything worked fine. In production with hundreds of concurrent requests, the counter was always lower than the actual request count.

I spent hours trying to figure out what was wrong. The database counts did not match the in-memory counter. I thought requests were being dropped. Then someone suggested running the tests with `-race`. Boom: "DATA RACE" on the counter variable.

The fix was a simple `atomic.AddInt64`. One line changed. Hours of debugging. That is when I truly learned: **always use the race detector. Always.** It is not optional. It is part of your development workflow.

Run your tests with `-race`:

```bash
go test -race ./...
```

Run your development server with `-race`:

```bash
go run -race main.go
```

Add it to your CI pipeline:

```bash
go build -race ./...
```

The race detector has some overhead (about 5-10x slower and uses more memory), but it is absolutely worth it for development and testing. Just do not use `-race` in production for performance reasons.

Race conditions are the silent killers of concurrent programs. The Go race detector is your shield. Use it.
