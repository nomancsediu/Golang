# Synchronization

## Why Synchronization Is Needed

When multiple goroutines access the same data, you need **synchronization** to prevent race conditions. A race condition happens when the result depends on the order of execution, and the order is unpredictable.

```go
// RACE CONDITION: unsafe counter
var counter int

func increment() {
    for i := 0; i < 1000; i++ {
        counter++ // Not safe! Multiple goroutines can read/write at the same time
    }
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    wg.Wait()
    fmt.Println(counter) // Should be 10000, but is usually less
}
```

Run with race detector to find these bugs:

```bash
go run -race main.go
```

## Synchronization Tools

### Mutex

A **mutex** (mutual exclusion lock) ensures only one goroutine can access a resource at a time:

```go
type SafeCounter struct {
    mu      sync.Mutex
    counter int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counter++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.counter
}
```

**Rules for mutex**:
- Always unlock with `defer` to prevent deadlocks
- Keep the locked section as small as possible
- Never lock a mutex twice in the same goroutine (that causes a deadlock)

### RWMutex

An **RWMutex** allows multiple readers or one writer. Use it when reads are frequent and writes are rare:

```go
type SafeCache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()         // Multiple readers can hold RLock simultaneously
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()          // Only one writer at a time, blocks all readers
    defer c.mu.Unlock()
    c.items[key] = value
}
```

RWMutex is great for caches, configurations, and other read-heavy data.

### WaitGroup

A **WaitGroup** waits for a group of goroutines to finish:

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1) // Add before starting the goroutine
        go func(id int) {
            defer wg.Done()
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }

    wg.Wait() // Block until all goroutines call Done()
    fmt.Println("All workers finished")
}
```

**Common mistake**: calling `wg.Add(1)` inside the goroutine instead of before. This can cause `wg.Wait()` to return before all goroutines start.

### Once

**sync.Once** ensures a function runs exactly once, no matter how many goroutines call it:

```go
var instance *Database
var once sync.Once

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{}
        instance.Connect()
    })
    return instance
}
```

This is the standard way to implement lazy initialization in Go.

### Atomic Operations

The **sync/atomic** package provides low-level atomic operations. Use them for simple counters and flags:

```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}

func getValue() int64 {
    return atomic.LoadInt64(&counter)
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    wg.Wait()
    fmt.Println(atomic.LoadInt64(&counter)) // Always 10000
}
```

Atomic operations are faster than mutexes for simple operations. Use them for counters, flags, and pointer swaps.

### Channels

Channels are Go's preferred way to synchronize goroutines. Instead of sharing memory and protecting it with locks, you pass data through channels:

```go
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int, done chan<- bool) {
    for val := range ch {
        fmt.Println("Received:", val)
    }
    done <- true
}

func main() {
    ch := make(chan int)
    done := make(chan bool)

    go producer(ch)
    go consumer(ch, done)

    <-done // Wait for consumer to finish
}
```

## When to Use Each Tool

| Tool | Use Case | Performance |
|------|----------|-------------|
| **Channel** | Goroutine communication, pipelines | Moderate |
| **Mutex** | Protecting shared data, complex operations | Good |
| **RWMutex** | Read-heavy shared data | Good |
| **WaitGroup** | Waiting for multiple goroutines | Good |
| **Once** | One-time initialization | Good |
| **Atomic** | Simple counters, flags | Excellent |

My rule of thumb: **use channels when goroutines need to communicate, use mutexes when goroutines need to share data**. If you are just passing data from one goroutine to another, a channel is cleaner. If multiple goroutines need to read and write the same data structure, a mutex is simpler.

## Common Synchronization Patterns

### Protecting a Map

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}
```

### Or Channel Pattern

Wait for the first of multiple operations to complete:

```go
func fetchFromMultipleSources(ctx context.Context) ([]byte, error) {
    results := make(chan []byte, 3)
    errors := make(chan error, 3)

    // Try three sources simultaneously
    go func() { results <- fetchSource1(ctx) }()
    go func() { results <- fetchSource2(ctx) }()
    go func() { results <- fetchSource3(ctx) }()

    select {
    case data := <-results:
        return data, nil
    case err := <-errors:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

## Avoiding Common Pitfalls

**Deadlock** - Two goroutines waiting for each other's locks. Avoid by always locking in the same order and using `defer` to unlock.

**Race condition** - Two goroutines accessing data without synchronization. Use the race detector (`go run -race`).

**Starvation** - One goroutine never gets access to a resource. Use fair locking patterns and avoid holding locks for too long.

Synchronization is not optional in concurrent programming. Every shared resource must be protected. The Go race detector is your best friend for finding bugs. Run your tests with `-race` always.
