# WaitGroup in Go

## What is a WaitGroup?

When you start multiple goroutines, you often need to wait for all of them to finish before your program continues or exits. A **WaitGroup** is the simplest and most common way to do this in Go.

Think of it like a counter at a restaurant. You have 5 friends in the kitchen. You add 5 to the counter. Each friend decrements the counter when they finish eating. When the counter reaches zero, everyone is done and you can leave.

## Import and Basic Usage

WaitGroup lives in the `sync` package:

```go
import "sync"
```

It has three methods:

- **`Add(n int)`** - Add n to the counter (usually called before starting a goroutine)
- **`Done()`** - Decrement the counter by 1 (usually called with defer inside the goroutine)
- **`Wait()`** - Block until the counter reaches 0

## The Basic Pattern

Here is the pattern I use almost every time:

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // Decrement counter when function returns

    fmt.Printf("Worker %d starting\n", id)
    // Do some work here...
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)            // Increment counter before starting goroutine
        go worker(i, &wg)    // Pass WaitGroup pointer to goroutine
    }

    wg.Wait() // Block until all workers call Done()
    fmt.Println("All workers finished!")
}
```

The output might look like:

```
Worker 3 starting
Worker 1 starting
Worker 5 starting
Worker 2 starting
Worker 4 starting
Worker 3 done
Worker 1 done
Worker 5 done
Worker 2 done
Worker 4 done
All workers finished!
```

The order of "starting" and "done" messages varies because goroutines run concurrently. But "All workers finished!" always prints last because `wg.Wait()` ensures all goroutines complete before the program moves on.

## Why Pass a Pointer?

Notice that I pass `&wg` (a pointer) to the goroutine, not `wg` (a value). This is because **WaitGroup must not be copied after first use.** If you copy a WaitGroup, the counter inside the copy and the original become separate. Your `Done()` calls would decrement the copy, not the original, and `Wait()` would block forever.

The Go vet tool catches this mistake:

```bash
go vet ./...
# warns: call of wg.Done copies the WaitGroup value
```

## A Common Bug: Add Inside the Goroutine

This is a mistake I made early on and it took me a while to figure out:

```go
// WRONG! Race condition!
for i := 1; i <= 5; i++ {
    go func(id int) {
        wg.Add(1)         // Too late! Main might already call Wait()
        defer wg.Done()
        fmt.Printf("Worker %d\n", id)
    }(i)
}
wg.Wait() // Might return immediately if goroutines haven't started yet
```

The problem is that `wg.Add(1)` runs inside the goroutine, which starts asynchronously. By the time the goroutine gets scheduled and calls `Add(1)`, the main goroutine might have already called `wg.Wait()` and found the counter at zero. The program would exit before any goroutine does its work.

**Always call `Add` before starting the goroutine, not inside it.**

The correct way:

```go
for i := 1; i <= 5; i++ {
    wg.Add(1)             // Increment BEFORE starting goroutine
    go func(id int) {
        defer wg.Done()
        fmt.Printf("Worker %d\n", id)
    }(i)
}
wg.Wait()
```

## Adding Multiple at Once

You can also add multiple to the counter in a single call:

```go
var wg sync.WaitGroup
wg.Add(5) // Add 5 at once instead of calling Add(1) five times

for i := 1; i <= 5; i++ {
    go func(id int) {
        defer wg.Done()
        fmt.Printf("Worker %d\n", id)
    }(i)
}

wg.Wait()
```

This is useful when you know the exact number of goroutines upfront. It is slightly more efficient than calling `Add(1)` in a loop because it only acquires the internal lock once.

## WaitGroup with Multiple Workers

Here is a more realistic example with a worker pool pattern:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func processJob(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Millisecond * 500) // Simulate work
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int, 10)

    var wg sync.WaitGroup

    // Start 3 workers
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go processJob(w, jobs, results, &wg)
    }

    // Send 9 jobs
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs) // Signal workers that no more jobs are coming

    // Wait for all workers to finish, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }

    fmt.Println("All jobs processed!")
}
```

Notice the pattern: workers read from the `jobs` channel until it is closed, write to the `results` channel, and call `wg.Done()` when they finish. A separate goroutine waits for all workers and then closes `results` so the main goroutine can range over it.

## Real Example: Concurrent Web Scraper

Here is a practical example of using WaitGroup to scrape multiple web pages concurrently:

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type PageResult struct {
    URL   string
    Size  int
    Error error
}

func fetchPage(url string, results chan<- PageResult, wg *sync.WaitGroup) {
    defer wg.Done()

    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        results <- PageResult{URL: url, Error: err}
        return
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        results <- PageResult{URL: url, Error: err}
        return
    }

    results <- PageResult{
        URL:  url,
        Size: len(body),
    }
    fmt.Printf("Fetched %s in %v\n", url, time.Since(start))
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://google.com",
        "https://github.com",
        "https://stackoverflow.com",
        "https://go.dev",
    }

    results := make(chan PageResult, len(urls))
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go fetchPage(url, results, &wg)
    }

    // Wait in a separate goroutine, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect all results
    for result := range results {
        if result.Error != nil {
            fmt.Printf("Error fetching %s: %v\n", result.URL, result.Error)
        } else {
            fmt.Printf("%s: %d bytes\n", result.URL, result.Size)
        }
    }
}
```

This fetches all five URLs at the same time instead of one by one. The total time is roughly the time of the slowest single request, not the sum of all requests.

## My First Concurrency Tool

WaitGroup was my very first concurrency tool in Go. Before I understood channels, before I knew about mutexes, WaitGroup was there. It is simple, intuitive, and solves the most basic concurrency problem: "I started some things, now I need to wait for them."

Even now, after learning about channels, context, and other concurrency primitives, WaitGroup is still the tool I reach for first when I just need to wait for goroutines. It is the hammer in my concurrency toolbox. Not every problem is a nail, but many are.

The key things to remember: **call Add before the goroutine starts**, **always use defer Done()**, and **never copy a WaitGroup**. Get those three things right and WaitGroup will serve you well.
