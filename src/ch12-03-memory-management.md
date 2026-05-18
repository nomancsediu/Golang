# Memory Management

## Stack vs Heap (Recap)

Every Go program uses two areas of memory: the **stack** and the **heap**.

**Stack** - Fast, automatic memory. Each goroutine has its own stack. When a function returns, its stack frame is removed. No garbage collection needed.

**Heap** - Slower, dynamic memory. Data that lives beyond the function that created it goes on the heap. The garbage collector cleans up heap memory that is no longer used.

```go
func example() {
    // x lives on the stack (fast)
    x := 42

    // y escapes to the heap because we return a pointer to it
    y := allocateOnHeap()
    fmt.Println(x, y)
}

func allocateOnHeap() *int {
    y := 100
    return &y // y escapes to the heap
}
```

The general rule: if the compiler can prove a variable is only used within the function, it goes on the stack. If the variable might be used after the function returns (like returning a pointer), it goes on the heap.

## Go's Escape Analysis

The Go compiler performs **escape analysis** to decide whether a variable goes on the stack or the heap. You can see the decisions with:

```bash
go build -gcflags="-m" main.go
```

Output looks like:

```
./main.go:8:6: &y escapes to heap
./main.go:7:2: moved to heap: y
```

Common things that cause heap allocation:

- Returning a pointer to a local variable
- Storing a pointer in a struct that outlives the function
- Sending a value to a channel
- Storing a value in an interface
- Closures that capture variables

## Reducing Heap Allocations

Heap allocations are expensive. They require the garbage collector to track and clean up. In performance-critical code, reducing heap allocations can make a big difference.

### Use sync.Pool

**sync.Pool** reuses objects instead of creating new ones:

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) {
    // Get a buffer from the pool
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf) // Return to pool
    }()

    buf.Write(data)
    // Process buf...
}
```

Without the pool, every request creates a new buffer that goes on the heap and needs garbage collection. With the pool, buffers are reused.

### Preallocate Slices and Maps

When you know the size, preallocate:

```go
// Bad: slice grows multiple times, each growth allocates on the heap
var items []string
for i := 0; i < 1000; i++ {
    items = append(items, fmt.Sprintf("item-%d", i))
}

// Good: one allocation
items := make([]string, 0, 1000)
for i := 0; i < 1000; i++ {
    items = append(items, fmt.Sprintf("item-%d", i))
}
```

### Use Value Types When Possible

Passing value types (instead of pointers) avoids heap allocation when the value is small:

```go
// For small structs (a few fields), use values
type Point struct {
    X, Y float64
}

func distance(p Point) float64 { // value, stays on stack
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

// For large structs or when you need to modify, use pointers
type BigData struct {
    Records [10000]int
}

func process(d *BigData) { // pointer, avoids copying large struct
    // ...
}
```

## Memory Leaks in Go

Go has a garbage collector, so you might think memory leaks are impossible. They are not. A **memory leak** in Go happens when you hold references to objects you no longer need.

### Goroutine Leaks

The most common memory leak in Go is a **goroutine leak**. If a goroutine is blocked forever, it and everything it references stays in memory:

```go
// BAD: goroutine leak
func processData() <-chan int {
    ch := make(chan int)
    go func() {
        result := expensiveComputation()
        ch <- result // If nobody reads from ch, this goroutine blocks forever
    }()
    return ch
}

func handler(w http.ResponseWriter, r *http.Request) {
    ch := processData()
    select {
    case result := <-ch:
        fmt.Println(result)
    case <-time.After(1 * time.Second):
        // Timeout! But the goroutine in processData is still blocked
        fmt.Println("Timed out")
    }
}
```

The fix: use buffered channels or context cancellation:

```go
func processData(ctx context.Context) <-chan int {
    ch := make(chan int, 1) // Buffered channel
    go func() {
        select {
        case ch <- expensiveComputation():
        case <-ctx.Done():
            // Context cancelled, goroutine can exit
        }
    }()
    return ch
}
```

### Unclosed Resources

Forgetting to close HTTP response bodies, file handles, or database connections:

```go
// BAD: response body never closed
resp, _ := http.Get("https://example.com")
data, _ := io.ReadAll(resp.Body)
// resp.Body.Close() is missing!

// GOOD: always defer close
resp, err := http.Get("https://example.com")
if err != nil {
    return err
}
defer resp.Body.Close()
data, _ := io.ReadAll(resp.Body)
```

### Slice References

A slice that references a large underlying array:

```go
func loadData() []byte {
    data := make([]byte, 1000000) // 1MB
    return data[:100] // Returns a slice that references the entire 1MB array
}

// Fix: copy the data you need
func loadDataFixed() []byte {
    data := make([]byte, 1000000)
    result := make([]byte, 100)
    copy(result, data[:100])
    return result // Only 100 bytes, original 1MB can be garbage collected
}
```

## Profiling with pprof

Go has a built-in profiler called **pprof**. It shows you where your program uses memory and CPU:

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Your application...
}
```

Then analyze the profiles:

```bash
# Memory profile
go tool pprof http://localhost:6060/debug/pprof/heap

# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

Inside pprof:

```
(pprof) top 10     # Show top 10 memory consumers
(pprof) list func  # Show line-by-line allocation in func
(pprof) web        # Generate a visual graph
```

## Code Example: Finding and Fixing a Memory Leak

```go
// Simulated memory leak: global cache that never stops growing
var cache = make(map[string][]byte)

func handler(w http.ResponseWriter, r *http.Request) {
    data := generateData()
    cache[r.URL.Path] = data // Keeps growing forever
    w.Write(data)
}

// Fix: limit cache size with eviction
type LRUCache struct {
    items map[string]cacheEntry
    mu    sync.Mutex
    max   int
}

type cacheEntry struct {
    data      []byte
    accessed  time.Time
}

func (c *LRUCache) Set(key string, data []byte) {
    c.mu.Lock()
    defer c.mu.Unlock()

    // Evict old entries if cache is full
    if len(c.items) >= c.max {
        c.evictOldest()
    }

    c.items[key] = cacheEntry{data: data, accessed: time.Now()}
}
```

Understanding memory management in Go is not about memorizing rules. It is about being aware of where your data lives and making conscious choices about allocation. Profile before you optimize, but always write code with memory in mind.
