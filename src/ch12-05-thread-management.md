# Thread Management

## OS Threads vs Goroutines (Recap)

An **OS thread** is managed by the operating system. It has its own stack (usually 1-8 MB), and switching between threads involves a context switch that is relatively expensive.

A **goroutine** is managed by the Go runtime. It starts with a small stack (2 KB) that can grow as needed. Switching between goroutines is much cheaper than switching between OS threads because it is done in user space by the Go scheduler.

Key differences:

| Feature | OS Thread | Goroutine |
|---------|-----------|-----------|
| **Created by** | Operating system | Go runtime |
| **Stack size** | 1-8 MB (fixed) | 2 KB (grows dynamically) |
| **Creation cost** | Expensive | Cheap |
| **Switch cost** | Expensive (kernel) | Cheap (user space) |
| **Typical count** | Thousands | Millions |

You can easily run hundreds of thousands of goroutines. You cannot run that many OS threads.

## The Go Scheduler (GMP Model)

The Go scheduler uses the **GMP model**:

- **G** (Goroutine) - The goroutine being scheduled
- **M** (Machine) - An OS thread
- **P** (Processor) - A logical processor that runs goroutines

How it works:

1. Each **P** has a queue of goroutines waiting to run.
2. A **P** executes goroutines on an **M** (OS thread).
3. When a goroutine blocks (I/O, channel operation, sleep), the P detaches from that M and picks up another M to continue running other goroutines.
4. When the blocked goroutine unblocks, it tries to get a P to run on. If none is available, it goes on a global queue.

The number of **P**s defaults to the number of CPU cores. This is controlled by `GOMAXPROCS`.

## runtime.GOMAXPROCS

The `GOMAXPROCS` setting controls how many OS threads can execute Go code simultaneously:

```go
func main() {
    // Get current setting
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))

    // Set to a specific value
    runtime.GOMAXPROCS(4)

    // Set to number of CPU cores (default)
    runtime.GOMAXPROCS(runtime.NumCPU())
}
```

Before Go 1.5, GOMAXPROCS defaulted to 1. Since Go 1.5, it defaults to the number of available CPU cores. You rarely need to change it.

## Limiting Goroutines

Just because you can create millions of goroutines does not mean you should. Unbounded goroutine creation can exhaust memory and overwhelm downstream services.

### Using a Buffered Channel as a Semaphore

```go
func processAll(items []Item) {
    // Limit to 10 concurrent goroutines
    sem := make(chan struct{}, 10)
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(i Item) {
            defer wg.Done()
            sem <- struct{}{} // Acquire semaphore
            defer func() { <-sem }() // Release semaphore

            processItem(i)
        }(item)
    }

    wg.Wait()
}
```

### Using a Worker Pool

The worker pool pattern is more structured:

```go
func processAll(items []Item, numWorkers int) {
    jobs := make(chan Item, len(items))
    var wg sync.WaitGroup

    // Start workers
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                processItem(item)
            }
        }()
    }

    // Send jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    wg.Wait()
}
```

## Worker Pool Pattern

A reusable worker pool implementation:

```go
package pool

import "sync"

type Pool struct {
    tasks   chan Task
    workers int
    wg      sync.WaitGroup
}

type Task func()

func New(workers int, bufferSize int) *Pool {
    return &Pool{
        tasks:   make(chan Task, bufferSize),
        workers: workers,
    }
}

func (p *Pool) Start() {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go func() {
            defer p.wg.Done()
            for task := range p.tasks {
                task()
            }
        }()
    }
}

func (p *Pool) Submit(task Task) {
    p.tasks <- task
}

func (p *Pool) Stop() {
    close(p.tasks)
    p.wg.Wait()
}
```

Usage:

```go
func main() {
    pool := pool.New(10, 100)
    pool.Start()

    for i := 0; i < 1000; i++ {
        id := i
        pool.Submit(func() {
            fmt.Printf("Processing task %d\n", id)
        })
    }

    pool.Stop()
}
```

## Graceful Shutdown with Signal Handling

A production server must handle shutdown gracefully. When you press Ctrl+C or send a termination signal, the server should:

1. Stop accepting new requests
2. Finish processing current requests
3. Close database connections
4. Then exit

```go
func main() {
    // Create a server
    srv := &http.Server{
        Addr:    ":8080",
        Handler: routes(),
    }

    // Start the server in a goroutine
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Printf("Received signal: %v, shutting down...\n", sig)

    // Give outstanding requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("Server forced to shutdown: %v", err)
    }

    // Close database connections
    db.Close()

    log.Println("Server stopped gracefully")
}
```

The `srv.Shutdown()` method gracefully shuts down the server without interrupting active connections. It first closes the listener, then waits for all active requests to complete, or the timeout to expire.

## Monitoring Goroutines

Use the runtime package to monitor goroutine count:

```go
func monitorGoroutines() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        log.Printf("Goroutines: %d", runtime.NumGoroutine())
    }
}

func main() {
    go monitorGoroutines()
    // ... rest of your application
}
```

If the goroutine count keeps growing, you likely have a goroutine leak. Investigate with pprof:

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

## My Note on Thread Management

I used to create a new goroutine for every task without thinking. In development, it worked fine. In production with thousands of concurrent requests, it became a problem. Too many goroutines fighting for resources. Too many database connections.

The worker pool pattern fixed this. By limiting concurrency, I made my application predictable and stable. The rule I follow now: **always limit concurrency**. Use a worker pool or semaphore. Never let goroutine creation be unbounded.

Graceful shutdown was another lesson I learned the hard way. My first production deployment, I just killed the process. In-flight requests were cut off. Database connections were left dangling. After adding graceful shutdown, deployments became smooth. Zero dropped requests.
