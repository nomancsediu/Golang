# Map Usage in Go

## What is a Map?

A **map** is Go's built-in key-value data structure. It is similar to a dictionary in Python, a hash map in Java, or an object in JavaScript. You use a key to look up a value.

Maps are one of the most useful data structures in Go. I use them every day for lookups, counting, grouping, and configuration.

## Creating a Map

There are several ways to create a map:

```go
// Using make
ages := make(map[string]int)

// Using a literal
ages := map[string]int{
    "Alice": 30,
    "Bob":   25,
    "Carol": 35,
}

// Empty map literal (different from nil map)
scores := map[string]int{}
```

**Important**: A nil map behaves differently from an empty map. A nil map returns zero values when read but panics when you try to write. Always initialize maps with `make` or a literal before writing:

```go
var m map[string]int
m["key"] = 1 // PANIC! assignment to entry in nil map

m = make(map[string]int)
m["key"] = 1 // OK
```

## Adding and Updating

```go
m := make(map[string]int)

// Add a key-value pair
m["Alice"] = 30

// Update an existing key
m["Alice"] = 31

// Add more
m["Bob"] = 25
m["Carol"] = 35
```

## Deleting

```go
delete(m, "Bob") // Remove Bob from the map

// Delete is safe even if the key does not exist
delete(m, "NonExistent") // No error, no panic
```

## Checking Existence

When you access a key that does not exist, Go returns the **zero value** for the value type. To distinguish between "key exists with zero value" and "key does not exist," use the two-value form:

```go
m := map[string]int{"Alice": 30, "Bob": 0}

// One-value form: cannot tell if Bob exists with value 0 or does not exist
value := m["Bob"]    // 0
value = m["Unknown"] // 0 (also 0, but key does not exist)

// Two-value form: ok tells you if the key exists
value, ok := m["Bob"]     // value=0, ok=true
value, ok = m["Unknown"]  // value=0, ok=false

// Common pattern: check existence
if value, ok := m["Alice"]; ok {
    fmt.Printf("Alice's age is %d\n", value)
}
```

## Iterating with Range

```go
m := map[string]int{
    "Alice": 30,
    "Bob":   25,
    "Carol": 35,
}

for key, value := range m {
    fmt.Printf("%s is %d years old\n", key, value)
}

// Only keys
for key := range m {
    fmt.Println(key)
}

// Only values
for _, value := range m {
    fmt.Println(value)
}
```

**Important**: Map iteration order is **random**. Go intentionally randomizes the order so you do not accidentally depend on it. If you need a specific order, sort the keys first:

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
```

## Maps Are Reference Types

When you assign a map to another variable or pass it to a function, you are passing a reference to the same underlying data. Changes in one are visible in the other:

```go
m := map[string]int{"Alice": 30}
n := m         // n points to the same map
n["Bob"] = 25  // m["Bob"] is now also 25

func addEntry(m map[string]int) {
    m["Carol"] = 35 // This modifies the original map
}
```

This is different from slices, where appending can create a new underlying array. With maps, there is no "copy on write." It is always the same map.

## Maps Are NOT Concurrent-Safe

If multiple goroutines read and write to the same map at the same time, your program will panic. This is one of the most common concurrency bugs in Go:

```go
m := make(map[string]int)

// DANGEROUS: concurrent writes cause panic
go func() { m["key1"] = 1 }()
go func() { m["key2"] = 2 }() // PANIC: concurrent map writes
```

### Solution 1: sync.Mutex

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

func (sm *SafeMap) Delete(key string) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    delete(sm.m, key)
}
```

### Solution 2: sync.Map

Go provides **sync.Map** for concurrent maps. It is optimized for cases where keys are written once and read many times:

```go
var m sync.Map

// Store a value
m.Store("Alice", 30)

// Load a value
val, ok := m.Load("Alice")

// Load or Store (atomic)
actual, loaded := m.LoadOrStore("Bob", 25)

// Delete
m.Delete("Alice")

// Range over all entries
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%s: %v\n", key, value)
    return true // return false to stop iteration
})
```

**sync.Map vs mutex-protected map**: Use sync.Map when keys are stable (written once, read often). Use a mutex-protected map when reads and writes are mixed.

## Common Map Patterns

### Lookup Table

```go
statusText := map[int]string{
    200: "OK",
    201: "Created",
    400: "Bad Request",
    401: "Unauthorized",
    404: "Not Found",
    500: "Internal Server Error",
}

text := statusText[404] // "Not Found"
```

### Counting

```go
func countWords(words []string) map[string]int {
    counts := make(map[string]int)
    for _, word := range words {
        counts[word]++
    }
    return counts
}

words := []string{"go", "is", "great", "go", "is", "fun"}
counts := countWords(words)
// counts = map[go:2 is:2 great:1 fun:1]
```

### Grouping

```go
func groupByCategory(products []Product) map[string][]Product {
    groups := make(map[string][]Product)
    for _, p := range products {
        groups[p.Category] = append(groups[p.Category], p)
    }
    return groups
}
```

### Set (unique values)

Go does not have a built-in set type. Use a map with empty struct values:

```go
type Set struct {
    items map[string]struct{}
}

func NewSet() *Set {
    return &Set{items: make(map[string]struct{})}
}

func (s *Set) Add(item string) {
    s.items[item] = struct{}{}
}

func (s *Set) Contains(item string) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set) Remove(item string) {
    delete(s.items, item)
}

func (s *Set) Size() int {
    return len(s.items)
}
```

Using `struct{}` as the value type means each entry uses zero bytes for the value. It is more memory-efficient than using `bool`.

## Getting the Length

```go
m := map[string]int{"Alice": 30, "Bob": 25}
fmt.Println(len(m)) // 2
```

## Maps in JSON

Maps with string keys encode naturally to JSON objects:

```go
m := map[string]int{"Alice": 30, "Bob": 25}
data, _ := json.Marshal(m)
fmt.Println(string(data)) // {"Alice":30,"Bob":25}
```

Maps with integer keys also work but produce string keys in JSON:

```go
m := map[int]string{1: "one", 2: "two"}
data, _ := json.Marshal(m)
fmt.Println(string(data)) // {"1":"one","2":"two"}
```

## My Note on Maps

Maps are one of those things that seem simple but have depth. I used maps for years before learning about the concurrency issues. The first time my program panicked with "concurrent map writes," I was confused for hours. Now I always think about concurrency when using maps.

The most useful pattern for me is the **lookup table**. Instead of long if-else chains, I put the mappings in a map and look them up by key. It is cleaner, easier to extend, and sometimes faster.

Maps are everywhere in Go. Understanding them well makes you a better Go programmer.
