# Goroutines in Go

## What is a Goroutine?

A **goroutine** is a lightweight thread of execution managed by the Go runtime. Notice I said "Go runtime," not "operating system." This is a crucial distinction. In languages like Java or C++, threads are managed by the OS. Each OS thread costs about 1MB of stack memory and takes significant time to create. Goroutines are different.

A goroutine starts with only about **2KB of stack space**, and that stack can grow and shrink as needed. Because they are so small, you can have hundreds of thousands of goroutines running at the same time. The Go runtime handles scheduling them onto real OS threads, so you never have to think about that yourself.

Think of it this way: an OS thread is like a big truck. Expensive to start, takes a lot of fuel, but carries a lot. A goroutine is like a bicycle. Cheap, fast to start, and you can have a whole lot of them on the road at once.

## Starting a Goroutine

Starting a goroutine is absurdly simple. You just add the keyword `go` before a function call:

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from goroutine!")
}

func main() {
    go sayHello()          // This runs as a goroutine
    fmt.Println("Hello from main!")

    time.Sleep(time.Second) // Wait so we can see the output
}
```

When you put `go` before a function call, Go starts a new goroutine and immediately moves to the next line. The function runs concurrently with the rest of your code. The `time.Sleep` is there just so the program does not exit before the goroutine has a chance to print. We will learn better ways to wait later.

## The Main Goroutine

When your Go program starts, the `main` function runs in what we call the **main goroutine**. This is important because **when the main goroutine finishes, the entire program exits, even if other goroutines are still running.**

```go
package main

import "fmt"

func sayHello() {
    fmt.Println("Hello from goroutine!")
}

func main() {
    go sayHello()
    fmt.Println("Hello from main!")
    // Program exits here. sayHello might not get to print!
}
```

In this example, there is a good chance "Hello from goroutine!" never gets printed. The main function finishes, the program exits, and the other goroutine is killed. This was one of the first concurrency gotchas I learned the hard way.

## Anonymous Goroutines

You do not need a named function to start a goroutine. You can use an **anonymous function** (also called a function literal):

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go func() {
        fmt.Println("Hello from anonymous goroutine!")
    }()

    fmt.Println("Hello from main!")
    time.Sleep(time.Second)
}
```

Notice the `()` at the end of the anonymous function. That is what actually calls the function. Without it, you are just defining a function but never executing it. I made this mistake more than once.

You can also pass arguments to anonymous goroutines:

```go
go func(name string) {
    fmt.Println("Hello,", name)
}("Noman")
```

This is useful when you want each goroutine to work with different data.

## Goroutines Are Cheap

Let me show you the wow moment I had. Here is a program that spawns 100,000 goroutines:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup

    start := time.Now()

    for i := 0; i < 100000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Each goroutine does a tiny bit of work
            _ = id * 2
        }(i)
    }

    wg.Wait()
    fmt.Printf("100,000 goroutines finished in %v\n", time.Since(start))
}
```

On my machine, this runs in about 30 milliseconds. 100,000 goroutines in 30ms! Try that with OS threads and your machine will probably crash or run out of memory. This is why Go is so popular for high-concurrency servers.

## Common Mistake: Goroutine Leak

A **goroutine leak** happens when you start a goroutine that can never finish. It just hangs around forever, consuming memory. This is one of the sneakiest bugs in Go:

```go
package main

import (
    "fmt"
    "time"
)

func leakyFunction() {
    ch := make(chan int)
    go func() {
        val := <-ch // Nobody ever sends to this channel!
        fmt.Println(val)
    }()
    // The goroutine above will wait forever
}
```

In this example, the goroutine is waiting to receive from a channel that nobody ever sends to. It will never finish. If you call this function many times, you will accumulate more and more stuck goroutines, slowly eating up memory.

The fix is to always make sure there is a way for goroutines to finish. Use context cancellation, close channels, or add timeouts.

## Example: Concurrent HTTP Requests

Here is a practical example. Let us fetch data from multiple URLs at the same time:

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

func fetchURL(url string, wg *sync.WaitGroup) {
    defer wg.Done()

    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        fmt.Printf("Error fetching %s: %v\n", url, err)
        return
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)

    fmt.Printf("Fetched %s in %v (%d bytes)\n", url, time.Since(start), len(body))
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://google.com",
        "https://github.com",
    }

    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, &wg)
    }

    wg.Wait()
    fmt.Println("All requests completed!")
}
```

Instead of fetching each URL one after another (which would take the sum of all request times), we fetch them all at once (which takes only as long as the slowest one). This is the power of goroutines.

## Example: Parallel Processing

Another common pattern is splitting work across multiple goroutines:

```go
package main

import (
    "fmt"
    "sync"
)

func processChunk(data []int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    sum := 0
    for _, v := range data {
        sum += v
    }
    results <- sum
}

func main() {
    // Split data into chunks
    data := make([]int, 1000)
    for i := range data {
        data[i] = i + 1
    }

    numWorkers := 4
    chunkSize := len(data) / numWorkers
    results := make(chan int, numWorkers)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        start := i * chunkSize
        end := start + chunkSize
        if i == numWorkers-1 {
            end = len(data) // Handle remainder
        }

        wg.Add(1)
        go processChunk(data[start:end], results, &wg)
    }

    // Close results channel when all workers are done
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    total := 0
    for partial := range results {
        total += partial
    }

    fmt.Printf("Total sum: %d\n", total)
}
```

## My Wow Moment

The first time I ran a program with 100,000 goroutines, I just sat there staring at the output. It finished almost instantly. No thread pool configuration, no stack size tuning, no operating system limits to worry about. Just `go func()` and it works.

That is the magic of goroutines. They make concurrency feel natural instead of scary. But remember: easy to start does not mean easy to get right. We still need to learn about channels, synchronization, and avoiding bugs. Let us keep going.
