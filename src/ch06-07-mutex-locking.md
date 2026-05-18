# Mutex and Locking in Go

## What is a Mutex?

A **mutex** (mutual exclusion lock) is the most basic synchronization primitive. It ensures that only one goroutine can access a section of code at a time. When one goroutine holds the lock, all other goroutines that try to acquire it must wait.

Think of a mutex like a bathroom lock. Only one person can be inside at a time. Others wait outside until the person inside unlocks the door and comes out.

Go provides mutexes in the `sync` package:

- **`sync.Mutex`** - A basic mutual exclusion lock
- **`sync.RWMutex`** - A reader-writer lock that allows multiple readers or one writer

## sync.Mutex: Lock and Unlock

The basic `sync.Mutex` has two methods:

- **`Lock()`** - Acquire the lock. If another goroutine holds it, block until it is released.
- **`Unlock()`** - Release the lock so other goroutines can acquire it.

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu      sync.Mutex
    counter int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    c.counter++
    c.mu.Unlock()
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.counter
}

func main() {
    sc := SafeCounter{}
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            sc.Increment()
        }()
    }

    wg.Wait()
    fmt.Printf("Final counter: %d\n", sc.Value())
}
```

Output: `Final counter: 1000`. Every time. No race conditions.

## Always Defer Unlock

This is one of the most important rules with mutexes: **always use `defer` to unlock.**

```go
// GOOD: Unlock with defer
func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counter++
}

// BAD: Manual unlock (easy to forget if there are multiple return paths)
func (c *SafeCounter) Increment() {
    c.mu.Lock()
    if someCondition {
        c.mu.Unlock() // Easy to forget!
        return
    }
    c.counter++
    c.mu.Unlock() // Easy to forget!
}
```

With `defer`, the unlock happens no matter how the function returns: normally, with an early return, or even with a panic. Without `defer`, every return path must manually call Unlock, and forgetting even one creates a deadlock.

## sync.RWMutex: Readers and Writers

Sometimes you have data that is read frequently but written rarely. A regular mutex forces all readers to wait for each other, even though reading is safe in parallel. That is where **`sync.RWMutex`** helps:

- **`RLock()`** / **`RUnlock()`** - For readers. Multiple readers can hold RLock at the same time.
- **`Lock()`** / **`Unlock()`** - For writers. Only one writer at a time, and no readers while writing.

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()         // Multiple readers can enter
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()          // Only one writer, no readers allowed
    defer c.mu.Unlock()
    c.items[key] = value
}

func main() {
    cache := SafeCache{
        items: make(map[string]string),
    }

    var wg sync.WaitGroup

    // Writers
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("key%d", i), fmt.Sprintf("value%d", i))
        }(i)
    }

    // Readers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            val, ok := cache.Get(fmt.Sprintf("key%d", i%5))
            fmt.Printf("Reader %d: key%d = %v (found: %v)\n", i, i%5, val, ok)
        }(i)
    }

    wg.Wait()
}
```

RWMutex is ideal for **caches**, **configuration**, and **read-heavy data structures** where reads far outnumber writes.

## When to Use Mutex vs Channels

This is one of the most debated topics in the Go community. Here is how I think about it:

**Use channels when:**
- You are passing ownership of data between goroutines
- You are coordinating and communicating between goroutines
- You are building pipelines or fan-out/fan-in patterns
- The data flow is a core part of your design

**Use mutex when:**
- You have shared state that multiple goroutines need to read and write
- The shared state is a simple cache or counter
- Protecting a small critical section of code
- The data flow is not the focus, just protecting access is

Go's official wiki says: **"prefer channels"** but that does not mean "always use channels." It means start with channels and use mutex when it is the simpler solution. For a shared counter or a shared cache, a mutex is perfectly fine and often simpler than a channel-based approach.

## Common Mistakes

### Forgetting to Unlock

```go
mu.Lock()
// Oops, forgot to unlock!
// Other goroutines are stuck forever.
```

Fix: always use `defer mu.Unlock()`.

### Double Locking (Deadlock)

```go
func bad() {
    mu.Lock()
    mu.Lock() // Deadlock! Cannot lock a mutex we already hold.
    // This goroutine is waiting for itself to unlock. It never will.
}
```

Go does not support **reentrant locks** (also called recursive mutexes). If a goroutine holds a lock and tries to lock it again, it deadlocks immediately. This is by design. Reentrant locks are considered a code smell because they often indicate unclear ownership of the lock.

### Locking Too Much

```go
func slow() {
    mu.Lock()
    defer mu.Unlock()

    // Doing I/O while holding the lock!
    // This blocks all other goroutines for a long time.
    resp, _ := http.Get("https://example.com")
    data, _ := io.ReadAll(resp.Body)
    cache["example"] = string(data)
}
```

Keep critical sections as small as possible. Lock, do the minimum work needed to safely update shared state, and unlock as quickly as you can. Do not do I/O, sleep, or long computations while holding a lock.

## sync.Once: Run Exactly Once

A related tool in the `sync` package is **`sync.Once`**. It ensures a function runs exactly one time, no matter how many goroutines call it:

```go
package main

import (
    "fmt"
    "sync"
)

type Config struct {
    value string
}

var (
    config     *Config
    once       sync.Once
)

func GetConfig() *Config {
    once.Do(func() {
        fmt.Println("Initializing config (only once!)")
        config = &Config{value: "production"}
    })
    return config
}

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            c := GetConfig()
            fmt.Printf("Goroutine %d: config = %s\n", id, c.value)
        }(i)
    }

    wg.Wait()
}
```

"Initializing config (only once!)" prints exactly once, even though 10 goroutines call GetConfig(). The first goroutine to call `once.Do` runs the function. All others block until it finishes, then they all continue without running the function again.

`sync.Once` is perfect for **lazy initialization**, **singleton patterns**, and **setup code** that must run exactly one time.

## Example: Thread-Safe Counter

Putting it all together, here is a complete thread-safe counter:

```go
package main

import (
    "fmt"
    "sync"
)

type Counter struct {
    mu    sync.RWMutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Decrement() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value--
}

func (c *Counter) Value() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.value
}

func (c *Counter) Reset() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value = 0
}

func main() {
    counter := Counter{}
    var wg sync.WaitGroup

    // 500 increments
    for i := 0; i < 500; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    // 200 decrements
    for i := 0; i < 200; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Decrement()
        }()
    }

    wg.Wait()
    fmt.Printf("Final value: %d (expected 300)\n", counter.Value())
}
```

## Example: Concurrent Cache

Here is a simple concurrent-safe cache using RWMutex:

```go
package main

import (
    "fmt"
    "sync"
)

type Cache struct {
    mu    sync.RWMutex
    items map[string]interface{}
}

func NewCache() *Cache {
    return &Cache{
        items: make(map[string]interface{}),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *Cache) Size() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.items)
}

func main() {
    cache := NewCache()
    var wg sync.WaitGroup

    // Concurrent writes
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("key%d", i), i*10)
        }(i)
    }

    // Concurrent reads
    for i := 0; i < 50; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            if val, ok := cache.Get(fmt.Sprintf("key%d", i)); ok {
                fmt.Printf("key%d = %v\n", i, val)
            }
        }(i)
    }

    wg.Wait()
    fmt.Printf("Cache size: %d\n", cache.Size())
}
```

## When I Choose Mutex Over Channels

My personal rule of thumb: if I am just protecting shared state and there is no meaningful data flow between goroutines, I use a mutex. It is simpler, more direct, and easier for other developers to understand.

I reach for channels when goroutines need to coordinate, when data ownership needs to transfer, or when I am building a pipeline. Channels are the right tool for communication patterns.

Mutex is the right tool for shared state protection. Both are valid. Both are important. Knowing when to use each is part of becoming a skilled Go developer.
