# Design Patterns in Go

## What Are Design Patterns?

**Design patterns** are reusable solutions to common problems in software design. They are not code you copy and paste. They are templates that you adapt to your specific situation.

The famous "Gang of Four" book defined 23 patterns. But not all of them are useful in Go. Go has its own idioms, and some classic patterns are better replaced with simpler Go code.

I used to think design patterns were overrated. Then I kept running into the same problems and kept writing the same solutions. That is when I realized: patterns are just names for things you are already doing, but done properly.

## Which Patterns Work Well in Go

Go favors **composition over inheritance**. Patterns based on inheritance (like Template Method or Visitor) do not fit naturally. Patterns based on composition and interfaces (like Strategy, Adapter, Factory) work beautifully.

Let me cover the patterns I use most often in Go.

## 1. Singleton (with sync.Once)

The **Singleton** pattern ensures a type has only one instance. In Go, the idiomatic way is using **sync.Once**:

```go
package config

import (
    "sync"
)

type Config struct {
    DatabaseURL string
    Port        int
}

var (
    instance *Config
    once     sync.Once
)

// GetConfig returns the single Config instance
func GetConfig() *Config {
    once.Do(func() {
        // This runs only once, no matter how many goroutines call GetConfig
        instance = &Config{
            DatabaseURL: "postgres://localhost/mydb",
            Port:        8080,
        }
    })
    return instance
}
```

`sync.Once` guarantees that the initialization function runs exactly once, even if multiple goroutines call `GetConfig()` at the same time. No mutex needed. No race conditions.

Use singletons for things that truly need one instance: configuration, logger, database connection pool.

## 2. Factory Pattern

The **Factory** pattern creates objects without exposing the creation logic. In Go, this is usually a function that returns an interface:

```go
package storage

// The interface all storage implementations satisfy
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

// Factory function that creates the right storage
func NewStorage(storageType string) (Storage, error) {
    switch storageType {
    case "memory":
        return &MemoryStorage{data: make(map[string][]byte)}, nil
    case "file":
        return &FileStorage{basePath: "/tmp/storage"}, nil
    case "s3":
        return &S3Storage{bucket: "my-bucket"}, nil
    default:
        return nil, fmt.Errorf("unknown storage type: %s", storageType)
    }
}

// Memory implementation
type MemoryStorage struct {
    data map[string][]byte
}

func (s *MemoryStorage) Save(key string, data []byte) error {
    s.data[key] = data
    return nil
}

func (s *MemoryStorage) Load(key string) ([]byte, error) {
    return s.data[key], nil
}
```

The caller does not need to know about `MemoryStorage` or `S3Storage`. They just use `Storage`. The factory decides which one to create based on configuration.

## 3. Strategy Pattern

The **Strategy** pattern lets you switch algorithms at runtime. You define an interface, implement multiple strategies, and choose which one to use:

```go
package shipping

// The strategy interface
type ShippingStrategy interface {
    Calculate(weight float64) float64
}

// Standard shipping
type StandardShipping struct{}

func (s StandardShipping) Calculate(weight float64) float64 {
    return weight * 1.5 + 5.0
}

// Express shipping
type ExpressShipping struct{}

func (s ExpressShipping) Calculate(weight float64) float64 {
    return weight * 3.0 + 10.0
}

// Free shipping (promotion)
type FreeShipping struct{}

func (s FreeShipping) Calculate(weight float64) float64 {
    return 0.0
}

// Order uses the strategy
type Order struct {
    Items    []Item
    Strategy ShippingStrategy
}

func (o *Order) CalculateShipping() float64 {
    totalWeight := 0.0
    for _, item := range o.Items {
        totalWeight += item.Weight
    }
    return o.Strategy.Calculate(totalWeight)
}

// Usage
func main() {
    order := Order{
        Items:    []Item{{Name: "Book", Weight: 0.5}},
        Strategy: ExpressShipping{},
    }
    cost := order.CalculateShipping()
    fmt.Printf("Shipping cost: $%.2f\n", cost)
}
```

Adding a new shipping method is easy: just implement the `ShippingStrategy` interface. No existing code needs to change.

## 4. Adapter Pattern

The **Adapter** pattern converts one interface into another. This is useful when you want to use a library that has a different interface than what your code expects:

```go
package logger

// Our application's logger interface
type Logger interface {
    Info(msg string)
    Error(msg string)
}

// A third-party logger with a different interface
type ThirdPartyLogger struct{}

func (l *ThirdPartyLogger) Log(level string, message string) {
    fmt.Printf("[%s] %s\n", level, message)
}

// Adapter that makes ThirdPartyLogger work with our Logger interface
type LoggerAdapter struct {
    logger *ThirdPartyLogger
}

func (a *LoggerAdapter) Info(msg string) {
    a.logger.Log("INFO", msg)
}

func (a *LoggerAdapter) Error(msg string) {
    a.logger.Log("ERROR", msg)
}

// Usage
func main() {
    thirdParty := &ThirdPartyLogger{}
    var log Logger = &LoggerAdapter{logger: thirdParty}

    log.Info("Server started")
    log.Error("Something went wrong")
}
```

The adapter wraps the third-party logger and translates our interface calls into its interface calls. Our code never touches the third-party API directly.

## 5. Observer Pattern with Channels

The **Observer** pattern notifies interested parties when something happens. In Go, we can use channels:

```go
package eventbus

type Event struct {
    Type    string
    Payload interface{}
}

type EventBus struct {
    subscribers map[string][]chan Event
    mu          sync.RWMutex
}

func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string][]chan Event),
    }
}

func (b *EventBus) Subscribe(eventType string) chan Event {
    b.mu.Lock()
    defer b.mu.Unlock()

    ch := make(chan Event, 10)
    b.subscribers[eventType] = append(b.subscribers[eventType], ch)
    return ch
}

func (b *EventBus) Publish(event Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()

    for _, ch := range b.subscribers[event.Type] {
        select {
        case ch <- event:
        default:
            // Skip if channel is full
        }
    }
}
```

## My Note on Design Patterns

Do not overuse patterns. I went through a phase where I tried to use a pattern for everything. Every class was a singleton, every function was a factory, every interaction was an observer. The code became harder to read, not easier.

Patterns are tools, not goals. Use them when they solve a real problem. If a simple function does the job, use a simple function. If a direct dependency is fine, do not add an interface just because you can.

The best code is the code that does not need a pattern name to be understood.
