# Remove Tight Coupling

## What is Tight Coupling?

**Tight coupling** happens when two components depend on each other's implementation details. If you change one, you have to change the other. If one breaks, the other breaks too.

Think of it like two pieces of a puzzle glued together. You cannot replace one piece without breaking the other. In code, this means your components are not independent. They are stuck together.

## The Problem with Tight Coupling

Tight coupling causes three big problems:

**Hard to test** - You cannot test one component in isolation because it drags in all its dependencies. Want to test the order service? You need a real database, a real email service, and a real payment gateway. That is not a unit test anymore.

**Hard to change** - Changing one component ripples across the codebase. You change the database schema and suddenly 20 files need updates. You change the email provider and you are editing code in places you forgot existed.

**Hard to reuse** - You cannot use a component in a different context because it is wired to specific implementations. Your user service only works with PostgreSQL, so you cannot use it with a different database.

## A Before Example: Tight Coupling

Here is what tight coupling looks like in Go:

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/lib/pq"
)

// OrderService is tightly coupled to PostgreSQL
type OrderService struct {
    // Directly depends on a concrete database connection
    db *sql.DB
}

func NewOrderService() *OrderService {
    // The service creates its own dependency
    db, err := sql.Open("postgres", "postgres://localhost/mydb")
    if err != nil {
        panic(err)
    }
    return &OrderService{db: db}
}

func (s *OrderService) CreateOrder(productID int, quantity int) error {
    // Directly writes SQL in the service
    _, err := s.db.Exec(
        "INSERT INTO orders (product_id, quantity) VALUES ($1, $2)",
        productID, quantity,
    )
    return err
}

func (s *OrderService) SendConfirmation(orderID int) error {
    // Directly depends on a specific email service
    // If we want to change the email provider, we change this code
    fmt.Printf("Sending email via SMTP for order %d\n", orderID)
    return nil
}
```

What is wrong here?

- `OrderService` creates its own database connection. It is stuck with PostgreSQL.
- `OrderService` directly uses `sql.DB`. We cannot swap in a mock for testing.
- `SendConfirmation` is hardcoded to use SMTP. We cannot switch to a different provider.
- Testing this service requires a real database.

## The Fix: Depend on Interfaces

The solution is to **depend on interfaces, not implementations**. This is called the **Dependency Inversion Principle**. High-level modules should not depend on low-level modules. Both should depend on abstractions.

Here is the decoupled version:

```go
package main

// Define interfaces for the dependencies
type OrderRepository interface {
    Save(productID int, quantity int) (int, error)
    FindByID(id int) (*Order, error)
}

type EmailService interface {
    SendConfirmation(orderID int) error
}

// OrderService depends on interfaces, not concrete types
type OrderService struct {
    repo  OrderRepository
    email EmailService
}

// Dependencies are injected from outside
func NewOrderService(repo OrderRepository, email EmailService) *OrderService {
    return &OrderService{
        repo:  repo,
        email: email,
    }
}

func (s *OrderService) CreateOrder(productID int, quantity int) error {
    _, err := s.repo.Save(productID, quantity)
    if err != nil {
        return fmt.Errorf("failed to create order: %w", err)
    }
    return nil
}
```

Now `OrderService` does not know or care about PostgreSQL, SMTP, or any specific implementation. It just knows that it needs something that can `Save` orders and something that can `SendConfirmation`.

## Dependency Injection

**Dependency injection** means giving a component its dependencies from the outside instead of creating them internally. In the example above, we inject `OrderRepository` and `EmailService` through the constructor.

There are three common ways to do dependency injection in Go:

**Constructor injection** (the example above) - Pass dependencies through `NewXxx()` functions. This is the most common and recommended approach.

**Method injection** - Pass dependencies as method parameters. Useful when the dependency varies per call.

**Struct injection** - Set dependencies as public struct fields. Simple but less safe.

## Real Implementations

Now we can create concrete implementations of our interfaces:

```go
// PostgreSQL implementation
type PostgresOrderRepository struct {
    db *sql.DB
}

func (r *PostgresOrderRepository) Save(productID int, quantity int) (int, error) {
    var id int
    err := r.db.QueryRow(
        "INSERT INTO orders (product_id, quantity) VALUES ($1, $2) RETURNING id",
        productID, quantity,
    ).Scan(&id)
    return id, err
}

// Mock implementation for testing
type MockOrderRepository struct {
    orders []Order
}

func (r *MockOrderRepository) Save(productID int, quantity int) (int, error) {
    id := len(r.orders) + 1
    r.orders = append(r.orders, Order{ID: id, ProductID: productID, Quantity: quantity})
    return id, nil
}
```

## Wiring It All Together

In `main()`, you connect the concrete implementations to the interfaces:

```go
func main() {
    // Create real dependencies
    db, _ := sql.Open("postgres", "postgres://localhost/mydb")
    repo := &PostgresOrderRepository{db: db}
    email := &SMTPEmailService{host: "smtp.example.com"}

    // Inject dependencies
    service := NewOrderService(repo, email)

    // Use the service
    err := service.CreateOrder(1, 5)
    if err != nil {
        fmt.Println("Error:", err)
    }
}
```

## My Personal Lesson

I used to import concrete types everywhere. Every service directly created its own database connection. Every handler directly created its own service. It felt simple at first, but as the project grew, it became a nightmare.

Learning to depend on interfaces changed everything. Now I can swap PostgreSQL for MySQL by changing one line in `main()`. I can test any service with a mock. I can add a caching layer without touching the service code.

The trick is simple: **if a component creates its own dependency, it is coupled to it. If a component receives its dependency, it is decoupled from it**.
