# Concurrent Programming

## Go Concurrency Model Recap

Go's concurrency model is built on three core concepts:

- **Goroutines** - Lightweight functions that run concurrently
- **Channels** - Typed pipes that goroutines use to communicate
- **Select** - A statement that waits on multiple channel operations

The motto: **"Do not communicate by sharing memory; share memory by communicating."**

This means instead of using locks to protect shared variables, you use channels to pass data between goroutines. This avoids many of the pitfalls of traditional concurrent programming.

## Common Concurrency Patterns

### Worker Pool

A **worker pool** limits the number of goroutines working on tasks at the same time. This prevents resource exhaustion:

```go
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- processJob(job)
            }
        }()
    }

    wg.Wait()
    close(results)
}

func processJob(job Job) Result {
    // Do the work
    return Result{ID: job.ID, Data: "processed"}
}
```

Usage:

```go
func main() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)

    // Start workers
    go workerPool(jobs, results, 5)

    // Send jobs
    for i := 0; i < 50; i++ {
        jobs <- Job{ID: i, Payload: fmt.Sprintf("task-%d", i)}
    }
    close(jobs)

    // Collect results
    for result := range results {
        fmt.Printf("Result: %+v\n", result)
    }
}
```

### Pipeline

A **pipeline** processes data through a series of stages. Each stage is a goroutine that reads from an input channel and writes to an output channel:

```go
// Stage 1: Generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: Square each number
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: Print results
func print(in <-chan int) {
    for n := range in {
        fmt.Println(n)
    }
}

// Wire it up
func main() {
    // generate -> square -> print
    nums := generate(1, 2, 3, 4, 5)
    squares := square(nums)
    print(squares)
}
```

### Fan-out / Fan-in

**Fan-out** distributes work across multiple goroutines. **Fan-in** collects the results:

```go
// Fan-out: start multiple workers reading from the same channel
func fanOut(in <-chan int, numWorkers int) []<-chan int {
    channels := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        channels[i] = square(in) // Each worker processes independently
    }
    return channels
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for n := range c {
                out <- n
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Context for Cancellation and Timeouts

The **context** package is essential for controlling goroutines. Use it to cancel operations, set deadlines, and pass request-scoped values:

### context.WithTimeout

```go
func fetchData(ctx context.Context) ([]byte, error) {
    // Create a timeout of 3 seconds
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/data", nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("request timed out")
        }
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}
```

### context.WithCancel

```go
func processWithCancel(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    results := make(chan string)

    // Start processing
    go func() {
        time.Sleep(5 * time.Second)
        results <- "done"
    }()

    select {
    case result := <-results:
        fmt.Println("Got result:", result)
        return nil
    case <-ctx.Done():
        return fmt.Errorf("processing cancelled: %w", ctx.Err())
    }
}
```

## Concurrent Data Processing Pipeline

Here is a complete example that combines multiple patterns:

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Record struct {
    ID   int
    Data string
}

type ProcessedRecord struct {
    ID     int
    Result string
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Stage 1: Generate records
    records := generateRecords(ctx, 100)

    // Stage 2: Process with 5 workers (fan-out)
    processed := processRecords(ctx, records, 5)

    // Stage 3: Collect results
    for result := range processed {
        fmt.Printf("Processed record %d: %s\n", result.ID, result.Result)
    }
}

func generateRecords(ctx context.Context, count int) <-chan Record {
    out := make(chan Record)
    go func() {
        defer close(out)
        for i := 0; i < count; i++ {
            select {
            case out <- Record{ID: i, Data: fmt.Sprintf("data-%d", i)}:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func processRecords(ctx context.Context, in <-chan Record, workers int) <-chan ProcessedRecord {
    out := make(chan ProcessedRecord)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for record := range in {
                select {
                case out <- ProcessedRecord{
                    ID:     record.ID,
                    Result: "processed-" + record.Data,
                }:
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

The key principle: **always check `ctx.Done()`** in your goroutines. Without it, goroutines cannot be stopped, and you get leaks.

Concurrent programming in Go is powerful but requires discipline. Always think about: how will this goroutine stop? What happens if the context is cancelled? How do I collect the results? Answer these questions and your concurrent code will be solid.
