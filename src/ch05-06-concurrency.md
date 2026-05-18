# Concurrency

## The Concept That Changed Everything for Me

Concurrency is the reason I fell in love with Go. Before Go, I struggled with writing concurrent programs in other languages. Threads were confusing, locks were easy to get wrong, and async/await made my brain hurt. But once I understood what concurrency actually means, everything became clearer.

Let me start from the beginning.

## What Is Concurrency?

**Concurrency** means dealing with multiple things at once. Notice I said "dealing with," not "doing." That is an important distinction. Concurrency is about **structure**, about breaking a problem into independent parts that can be managed separately.

A concurrent program has multiple tasks in progress at the same time, but they might not be executing at the exact same moment. They take turns, or they wait for each other, or they coordinate in some way.

Think about a chef in a kitchen:

- The chef puts water on to boil
- While waiting for the water, the chef chops vegetables
- When the water boils, the chef adds pasta
- While the pasta cooks, the chef makes the sauce
- When the pasta is done, the chef drains it and combines everything

The chef is one person (one CPU core) handling multiple tasks (concurrency). The chef is not cooking the pasta and making the sauce at the exact same instant. Instead, the chef switches between tasks, making progress on each one.

## Concurrency Is About Structure

This was a big realization for me. **Concurrency is not about speed. It is about structure.**

When you design a concurrent program, you are breaking the problem into independent pieces that can be managed separately. This makes your program:

- **More responsive** - The UI does not freeze while waiting for a network request
- **Better organized** - Each task is a separate unit of work
- **More efficient** - The CPU does not sit idle while waiting for I/O
- **Easier to reason about** - Each goroutine does one thing

Here is how a web server works concurrently:

```
Client 1 sends request  -->  +---------+
Client 2 sends request  -->  | Server  |
Client 3 sends request  -->  |         |
                             | Handle  | --> goroutine for client 1
                             | each    | --> goroutine for client 2
                             | request | --> goroutine for client 3
                             +---------+
                                  |
                                  v
                          Responses sent back
                          to each client
```

Each request gets its own goroutine. They are all in progress at the same time, managed independently. Some are waiting for database queries, some are waiting for file reads, some are computing. They are all making progress.

## Concurrency vs Parallelism: They Are Different!

This is one of the most misunderstood topics in programming. **Concurrency and parallelism are not the same thing.**

- **Concurrency** is about **dealing with** multiple things at once (structure)
- **Parallelism** is about **doing** multiple things at the exact same time (execution)

You can have concurrency without parallelism. A single-core CPU can run a concurrent program by rapidly switching between tasks. The tasks are all in progress, but only one is executing at any given moment.

You can have parallelism without concurrency. A program that uses SIMD instructions to process multiple data points at once is parallel but not concurrent (there is only one task).

And of course, you can have both. A concurrent program running on a multi-core CPU can execute multiple tasks at the same time.

```
Concurrency (single core, time-sharing):

Core 1: [Task A][Task B][Task A][Task B][Task A]
              ^       ^       ^       ^
         switch   switch   switch   switch

Parallelism (multiple cores, same time):

Core 1: [Task AAAAAAAAAAAAAAAAAAAAAAA]
Core 2: [Task BBBBBBBBBBBBBBBBBBBBBBB]
         ^--- both running at same time ---^

Concurrency + Parallelism:

Core 1: [Task A][Task C][Task A][Task C]
Core 2: [Task B][Task D][Task B][Task D]
```

Rob Pike, one of the creators of Go, said it best: "Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once." They are different ideas, and confusing them leads to bad program design.

## Concurrent Programs on a Single Core

A concurrent program can run perfectly well on a single CPU core. The OS (or the Go runtime) rapidly switches between the concurrent tasks, giving each one a slice of time. This is called **time-sharing** or **interleaving**.

For example, a web server on a single-core machine can handle hundreds of concurrent requests. Most of the time, each request is waiting for something: a database query, a file read, a network response. While one request waits, another can use the CPU. The server feels like it is handling everything at once, even though only one thing is executing at any given moment.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // This program is concurrent but runs on a single core
    runtime.GOMAXPROCS(1) // use only 1 CPU core

    // Task A: making coffee
    go func() {
        steps := []string{
            "Boiling water...",
            "Adding coffee grounds...",
            "Pouring water...",
            "Waiting for brew...",
            "Coffee is ready!",
        }
        for _, step := range steps {
            fmt.Printf("Coffee: %s\n", step)
            time.Sleep(200 * time.Millisecond)
        }
    }()

    // Task B: making toast
    go func() {
        steps := []string{
            "Putting bread in toaster...",
            "Toasting...",
            "Adding butter...",
            "Toast is ready!",
        }
        for _, step := range steps {
            fmt.Printf("Toast: %s\n", step)
            time.Sleep(200 * time.Millisecond)
        }
    }()

    // Give goroutines time to finish
    time.Sleep(3 * time.Second)
    fmt.Println("Breakfast is served!")
}
```

The output interleaves the two tasks. Both are making progress. Both are "in progress" at the same time. But on a single core, only one is actually executing at any given instant.

## Why Concurrency Matters

1. **Responsive applications** - A UI thread should not block while loading data. Concurrent code lets the UI stay responsive while background work happens.

2. **Efficient servers** - A web server that handles one request at a time is terrible. Concurrency lets it handle many requests simultaneously, with each one making progress.

3. **Better resource use** - When one task is waiting for I/O, another can use the CPU. This means less wasted time and more throughput.

4. **Natural program structure** - Some problems are naturally concurrent. A chat server, a game loop, a pipeline of data transformations. Concurrency lets you express these naturally.

## Go Was Designed for Concurrency

Go has first-class support for concurrency built into the language:

- **Goroutines** - Lightweight concurrent functions (just add `go` before a function call)
- **Channels** - Safe communication between goroutines
- **Select** - Wait on multiple channel operations
- **Sync package** - Mutexes, wait groups, and other synchronization primitives

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func fetchURL(url string, wg *sync.WaitGroup, results chan<- string) {
    defer wg.Done()
    // Simulate network request
    time.Sleep(time.Duration(100) * time.Millisecond)
    results <- fmt.Sprintf("Fetched %s", url)
}

func main() {
    urls := []string{
        "https://api.example.com/users",
        "https://api.example.com/posts",
        "https://api.example.com/comments",
        "https://api.example.com/likes",
    }

    var wg sync.WaitGroup
    results := make(chan string, len(urls))

    // Launch all requests concurrently
    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, &wg, results)
    }

    // Wait for all to finish, then close channel
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results as they arrive
    for result := range results {
        fmt.Println(result)
    }

    fmt.Println("All requests complete!")
}
```

All four requests start at the same time and run concurrently. The total time is about 100ms (the longest request), not 400ms (the sum of all requests). That is the power of concurrency.

## The Takeaway

Concurrency is about structure, not speed. It is about designing your program as a set of independent, manageable tasks. Go gives you the tools to express this structure naturally with goroutines and channels. Once you think concurrently, you start seeing problems differently, and your programs become better organized and more efficient.
