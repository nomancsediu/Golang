# Deadlocks in Go

## What is a Deadlock?

A **deadlock** is a situation where two or more goroutines are waiting on each other, and none of them can proceed. They are all stuck, forever. The program hangs, and nothing happens. No crash, no error, just silence.

Imagine two people at a narrow hallway. Each is waiting for the other to step aside so they can pass. Neither will move. They wait forever. That is a deadlock.

Deadlocks are different from race conditions. Race conditions produce wrong results. Deadlocks produce no results. The program just stops.

## The Four Conditions for Deadlock

For a deadlock to occur, four conditions must all be true at the same time. This is known as the **Coffman conditions**:

1. **Mutual exclusion**: At least one resource is held by only one goroutine at a time (a mutex, a channel)
2. **Hold and wait**: A goroutine holds at least one resource and is waiting for another
3. **No preemption**: Resources cannot be forcibly taken away from a goroutine
4. **Circular wait**: There is a circular chain of goroutines, each waiting for a resource held by the next one

All four must be present for a deadlock. Remove any one, and the deadlock cannot happen.

## Common Deadlock: Locking Two Mutexes in Different Order

The most common way to create a deadlock in Go is when two goroutines each need two locks but acquire them in different order:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var mu1 sync.Mutex
var mu2 sync.Mutex

func goroutineA() {
    mu1.Lock()
    fmt.Println("A: locked mu1, waiting for mu2")
    time.Sleep(time.Millisecond * 100) // Ensure the timing lines up

    mu2.Lock() // DEADLOCK! mu2 is held by goroutineB
    fmt.Println("A: locked mu2")
    mu2.Unlock()
    mu1.Unlock()
}

func goroutineB() {
    mu2.Lock()
    fmt.Println("B: locked mu2, waiting for mu1")
    time.Sleep(time.Millisecond * 100)

    mu1.Lock() // DEADLOCK! mu1 is held by goroutineA
    fmt.Println("B: locked mu1")
    mu1.Unlock()
    mu2.Unlock()
}

func main() {
    go goroutineA()
    go goroutineB()

    time.Sleep(time.Second * 5)
    fmt.Println("This might never print due to deadlock!")
}
```

Here is what happens:

```
Goroutine A: locks mu1, then tries to lock mu2
Goroutine B: locks mu2, then tries to lock mu1

A holds mu1, waiting for mu2
B holds mu2, waiting for mu1

Circular wait! Both wait forever.
```

### The Fix: Always Lock in the Same Order

```go
func goroutineA() {
    mu1.Lock()
    defer mu1.Unlock()

    mu2.Lock() // Same order: mu1 then mu2
    defer mu2.Unlock()

    fmt.Println("A: doing work")
}

func goroutineB() {
    mu1.Lock() // Same order: mu1 then mu2
    defer mu1.Unlock()

    mu2.Lock()
    defer mu2.Unlock()

    fmt.Println("B: doing work")
}
```

By always locking mutexes in the same order, you break the circular wait condition. Deadlock cannot happen.

## Common Deadlock: Channel Send Without Receive

Another common deadlock pattern is sending on a channel when nobody is receiving, or receiving when nobody is sending:

```go
package main

import "fmt"

func main() {
    ch := make(chan int) // Unbuffered channel

    ch <- 42 // DEADLOCK! Nobody is receiving
    // The program blocks here forever

    fmt.Println("This never prints")
}
```

With an unbuffered channel, the sender blocks until a receiver is ready. If there is no receiver, the sender blocks forever.

The same happens in reverse:

```go
func main() {
    ch := make(chan int)

    val := <-ch // DEADLOCK! Nobody is sending
    // The program blocks here forever

    fmt.Println(val)
}
```

### The Fix: Ensure Matching Send and Receive

```go
func main() {
    ch := make(chan int)

    go func() {
        ch <- 42 // Sender runs in a goroutine
    }()

    val := <-ch // Main goroutine receives
    fmt.Println(val) // Prints 42
}
```

Or use a buffered channel:

```go
func main() {
    ch := make(chan int, 1) // Buffer size 1

    ch <- 42 // Does not block! Buffer has space

    val := <-ch
    fmt.Println(val) // Prints 42
}
```

## Go Runtime Deadlock Detection

The Go runtime can detect some deadlocks. When all goroutines are asleep (blocked), the runtime prints a message and exits:

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /path/main.go:8 +0x67
```

This is helpful, but it only works when **all** goroutines are blocked. If even one goroutine is still running (even if it is just an infinite loop doing nothing useful), the runtime will not detect the deadlock.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    go func() {
        for {
            time.Sleep(time.Hour) // This goroutine is not blocked, just sleeping
            // Because of this, Go runtime will NOT detect the deadlock below
        }
    }()

    ch <- 42 // Nobody receiving. Real deadlock, but Go will not catch it.
    fmt.Println("Never prints")
}
```

So while the runtime deadlock detector is nice, it is not a substitute for careful programming.

## How to Prevent Deadlocks

### 1. Always Lock in the Same Order

This is the number one rule. If you need multiple locks, always acquire them in the same order everywhere in your code. Document the ordering and stick to it.

```go
// Convention: always lock mu1 before mu2
func operation1() {
    mu1.Lock()
    defer mu1.Unlock()
    mu2.Lock()
    defer mu2.Unlock()
    // ...
}

func operation2() {
    mu1.Lock() // Same order!
    defer mu1.Unlock()
    mu2.Lock()
    defer mu2.Unlock()
    // ...
}
```

### 2. Use Timeouts

Never wait forever. Use `select` with a timeout:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    go func() {
        time.Sleep(2 * time.Second)
        ch <- 42
    }()

    select {
    case val := <-ch:
        fmt.Println("Received:", val)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout! Giving up.")
        // Do something else instead of waiting forever
    }
}
```

### 3. Use Select with Default

If you do not want to block at all, use `select` with a `default` case:

```go
select {
case ch <- value:
    fmt.Println("Sent value")
default:
    fmt.Println("Channel is full, cannot send right now")
    // Handle the case where the channel is not ready
}
```

### 4. Avoid Nested Locks

The simplest way to avoid deadlocks is to avoid holding more than one lock at a time. Restructure your code so each goroutine only needs one lock at a time:

```go
// Instead of holding both locks:
func bad() {
    mu1.Lock()
    mu2.Lock()
    // ...
    mu2.Unlock()
    mu1.Unlock()
}

// Restructure to use one lock at a time:
func better() {
    mu1.Lock()
    data1 := readData1()
    mu1.Unlock()

    mu2.Lock()
    data2 := readData2()
    mu2.Unlock()

    process(data1, data2)
}
```

This is not always possible, but when it is, it eliminates the possibility of deadlock entirely.

### 5. Keep Critical Sections Small

The less time you hold a lock, the less chance of a deadlock. Do the minimum work inside a locked section:

```go
// BAD: Holding lock while doing slow I/O
func bad() {
    mu.Lock()
    data := fetchDataFromDB()    // Slow!
    process(data)
    mu.Unlock()
}

// BETTER: Lock only for the critical section
func better() {
    data := fetchDataFromDB()    // No lock needed for I/O
    mu.Lock()
    process(data)                // Only lock for shared state access
    mu.Unlock()
}
```

## My Deadlock Story

I once spent four hours debugging a deadlock in a web service. The service would occasionally freeze and stop responding to any requests. The freeze happened maybe once a week in production, never in development.

After adding logging, I found that two goroutines were stuck. One was waiting for a database query result on a channel, and the other was waiting to acquire a mutex that the first goroutine held. Classic circular wait.

The fix was simple: release the mutex before waiting on the channel. One line moved up. But finding it took hours because the deadlock only happened under specific timing conditions.

The lessons I learned:

- **Always use `defer` for unlocks**. My bug was caused by an unlock that happened too late because of a conditional path.
- **Keep critical sections as small as possible**. If I had released the lock earlier, the deadlock would never have happened.
- **Test with the race detector**. While `-race` does not detect deadlocks directly, it helps you think about synchronization more carefully.
- **Add timeouts**. If the channel receive had a timeout, the service would have recovered instead of hanging forever.

Deadlocks are scary because they make your program silently stop. But with careful design, consistent lock ordering, and timeouts, you can avoid almost all of them. And when they do happen, the Go runtime will often catch them and give you a useful stack trace. Learn to read those traces. They are your roadmap to the problem.
