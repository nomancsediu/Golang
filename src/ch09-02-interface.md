# Interface in Go

## What is an Interface?

An **interface** is a set of method signatures. It defines what a type can do, but not how it does it. Think of it as a contract. Any type that implements all the methods of an interface automatically satisfies that interface.

This is different from languages like Java where you explicitly say `class Dog implements Animal`. In Go, you just implement the methods and you are done. The compiler figures out the rest.

## Defining an Interface

Here is how you define an interface in Go:

```go
// Writer is anything that can write bytes
type Writer interface {
    Write([]byte) (int, error)
}

// Reader is anything that can read bytes
type Reader interface {
    Read([]byte) (int, error)
}

// Notifier is anything that can send a notification
type Notifier interface {
    Notify(message string) error
}
```

An interface with one method is called a **single-method interface**. Go encourages small, focused interfaces. The standard library is full of single-method interfaces like `io.Reader`, `io.Writer`, and `error`.

## Implicit Satisfaction

Go interfaces are **implicit**. There is no `implements` keyword. A type satisfies an interface simply by implementing all its methods:

```go
// ConsoleNotifier implements Notifier implicitly
type ConsoleNotifier struct{}

func (c ConsoleNotifier) Notify(message string) error {
    fmt.Println("[NOTIFICATION]", message)
    return nil
}

// EmailNotifier also implements Notifier implicitly
type EmailNotifier struct {
    address string
}

func (e EmailNotifier) Notify(message string) error {
    fmt.Printf("Sending email to %s: %s\n", e.address, message)
    return nil
}

// Both can be used wherever Notifier is expected
func sendMessage(n Notifier, msg string) {
    n.Notify(msg)
}

func main() {
    console := ConsoleNotifier{}
    email := EmailNotifier{address: "user@example.com"}

    sendMessage(console, "Hello from console")  // works
    sendMessage(email, "Hello from email")      // also works
}
```

Neither `ConsoleNotifier` nor `EmailNotifier` explicitly says it implements `Notifier`. They just have a `Notify` method, and that is enough. This is one of Go's most elegant features.

## The Empty Interface

The **empty interface** `interface{}` has no methods. Every type satisfies it. In Go 1.18+, you can use the alias `any` instead:

```go
// These are the same thing
var x interface{}
var y any

// You can assign anything to an empty interface
x = 42
x = "hello"
x = []int{1, 2, 3}
```

Use empty interfaces sparingly. They bypass Go's type system. Good use cases: parsing unknown JSON, generic containers. Bad use cases: function parameters that could be anything.

## Type Assertion

Sometimes you need to get the concrete value back from an interface. You use a **type assertion**:

```go
var n Notifier = ConsoleNotifier{}

// Assert that n is a ConsoleNotifier
console, ok := n.(ConsoleNotifier)
if ok {
    fmt.Println("It is a ConsoleNotifier:", console)
}

// If the assertion is wrong, ok will be false
email, ok := n.(EmailNotifier)
if !ok {
    fmt.Println("Not an EmailNotifier")
}
```

Always use the `ok` form. Without it, a wrong assertion causes a **panic**:

```go
// Dangerous! Panics if n is not EmailNotifier
email := n.(EmailNotifier)
```

## Type Switch

A **type switch** is like a regular switch but for interface types:

```go
func describe(i interface{}) {
    switch v := i.(type) {
    case string:
        fmt.Printf("String: %s (length %d)\n", v, len(v))
    case int:
        fmt.Printf("Integer: %d\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    case []string:
        fmt.Printf("String slice with %d items\n", len(v))
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}
```

## Interface Composition

Interfaces can be composed from other interfaces:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter combines Reader and Writer
type ReadWriter interface {
    Reader
    Writer
}

// Closer adds a Close method
type Closer interface {
    Close() error
}

// ReadWriteCloser combines all three
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

This is how the standard library builds complex interfaces from simple ones. **Small interfaces compose well.**

## Common Standard Library Interfaces

Go's standard library is built on interfaces. Here are the most important ones:

**io.Reader** - The most famous Go interface. Anything you can read from:
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

**io.Writer** - Anything you can write to. Files, network connections, buffers:
```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**error** - Yes, error is an interface:
```go
type error interface {
    Error() string
}
```

**fmt.Stringer** - Like toString() in other languages:
```go
type Stringer interface {
    String() string
}
```

## Using Interfaces for Testing

One of the most practical uses of interfaces is **mocking** in tests:

```go
// The interface we depend on
type UserRepository interface {
    GetByID(id int) (*User, error)
    Save(user *User) error
}

// Real implementation
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) GetByID(id int) (*User, error) {
    // actual database query
    return nil, nil
}

// Mock implementation for testing
type MockUserRepository struct {
    Users map[int]*User
    Err   error
}

func (r *MockUserRepository) GetByID(id int) (*User, error) {
    if r.Err != nil {
        return nil, r.Err
    }
    return r.Users[id], nil
}

// Test using the mock
func TestGetUser(t *testing.T) {
    mock := &MockUserRepository{
        Users: map[int]*User{
            1: {ID: 1, Name: "Alice"},
        },
    }

    service := NewUserService(mock)
    user, err := service.GetUser(1)

    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}
```

No real database needed. The test is fast, isolated, and deterministic.

## Interface Values

An interface value has two parts: a **type** and a **value**:

```go
var n Notifier = ConsoleNotifier{}
// type: ConsoleNotifier, value: ConsoleNotifier{}

var n2 Notifier
// type: nil, value: nil

// A nil interface is different from an interface with a nil value
var p *ConsoleNotifier = nil
var n3 Notifier = p
// type: *ConsoleNotifier, value: nil
// n3 is NOT nil! This is a common gotcha.
```

## My Note on Go Interfaces

Go's implicit interfaces are genius. In Java, you need to plan ahead and declare `implements` everywhere. In Go, you can define an interface after the fact. You can write a function that accepts an interface, and any existing type that happens to have the right methods will work.

This makes refactoring easy. You can extract an interface from a concrete type at any time without changing the original type. You can satisfy an interface from a different package. It is incredibly flexible.

The rule I follow: **define interfaces in the package that uses them, not the package that implements them**. The consumer defines what it needs. The provider just implements it.
