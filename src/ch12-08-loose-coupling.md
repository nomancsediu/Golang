# Loose Coupling

## What is Loose Coupling?

**Loose coupling** means components know as little as possible about each other. When one component changes, it should not force changes in other components. When one component breaks, it should not break others.

The opposite is **tight coupling**, where components depend on each other's implementation details. Change one, and you have to change them all.

Think of it like Lego bricks vs superglue. Lego bricks (loose coupling) can be connected and disconnected easily. Superglue (tight coupling) makes a permanent bond that is hard to undo.

## How to Achieve Loose Coupling

Three main techniques:

1. **Interfaces** - Depend on abstractions, not concrete types
2. **Dependency Injection** - Receive dependencies from outside
3. **Event-Driven Design** - Components communicate through events, not direct calls

## Tight Coupling vs Loose Coupling

### Tight Coupling Example

```go
package service

import (
    "database/sql"
    "net/smtp"
)

type OrderService struct {
    db *sql.DB // Directly depends on PostgreSQL
}

func NewOrderService() *OrderService {
    db, _ := sql.Open("postgres", "postgres://localhost/mydb")
    return &OrderService{db: db}
}

func (s *OrderService) CreateOrder(items []Item) error {
    // Directly uses database
    _, err := s.db.Exec("INSERT INTO orders ...")
    if err != nil {
        return err
    }

    // Directly sends email via SMTP
    smtp.SendMail("smtp.gmail.com:587", nil,
        "from@gmail.com", []string{"customer@example.com"}, []byte("Order confirmed"))

    // Directly calls payment gateway
    http.Post("https://payments.example.com/charge", "application/json", nil)

    return nil
}
```

Problems:
- Cannot test without a real PostgreSQL database
- Cannot change the email provider without modifying OrderService
- Cannot change the payment gateway without modifying OrderService
- If SMTP is down, the whole CreateOrder fails
- Hard to add logging, metrics, or retries

### Loose Coupling Example

```go
package service

// Define interfaces for each dependency
type OrderRepository interface {
    Save(items []Item) (int64, error)
}

type EmailSender interface {
    SendOrderConfirmation(orderID int64) error
}

type PaymentProcessor interface {
    Charge(orderID int64, amount float64) error
}

// OrderService depends on interfaces, not implementations
type OrderService struct {
    repo    OrderRepository
    email   EmailSender
    payment PaymentProcessor
}

func NewOrderService(
    repo OrderRepository,
    email EmailSender,
    payment PaymentProcessor,
) *OrderService {
    return &OrderService{
        repo:    repo,
        email:   email,
        payment: payment,
    }
}

func (s *OrderService) CreateOrder(items []Item) error {
    // Save order through interface
    orderID, err := s.repo.Save(items)
    if err != nil {
        return fmt.Errorf("failed to save order: %w", err)
    }

    // Send email through interface
    if err := s.email.SendOrderConfirmation(orderID); err != nil {
        // Log the error but do not fail the order
        log.Printf("Failed to send confirmation email: %v", err)
    }

    // Process payment through interface
    if err := s.payment.Charge(orderID, calculateTotal(items)); err != nil {
        return fmt.Errorf("payment failed: %w", err)
    }

    return nil
}
```

Now you can:
- Test with mock implementations
- Switch email providers by changing one line in main()
- Add a logging decorator without touching OrderService
- Use a fake payment processor for testing

## Define Interfaces in the Consumer Package

A common question: where should you define the interface? In the package that provides the implementation, or in the package that uses it?

In Go, the convention is: **define the interface in the consumer package**. This keeps the dependency direction correct and prevents unnecessary coupling:

```go
// GOOD: interface defined in the consumer
package order

type EmailSender interface {
    Send(to, subject, body string) error
}

type Service struct {
    email EmailSender // Only needs what OrderService actually uses
}

// BAD: interface defined in the provider
package email

type Sender interface {
    Send(to, subject, body string) error
    SendWithAttachment(to, subject, body string, files []string) error
    SendHTML(to, subject, htmlBody string) error
    SendBatch(recipients []string, subject, body string) error
}

package order

type Service struct {
    email email.Sender // Forces dependency on the email package
}
```

The consumer should define only the methods it needs. This follows the **Interface Segregation Principle**: no client should be forced to depend on methods it does not use.

## Wire Dependencies in main()

The `main()` function is where you connect concrete implementations to interfaces:

```go
package main

import (
    "database/sql"
    "myapp/adapter/email"
    "myapp/adapter/payment"
    pg "myapp/adapter/repository/postgres"
    "myapp/domain/order"
    "myapp/usecase"
)

func main() {
    db, _ := sql.Open("postgres", "postgres://localhost/mydb")
    defer db.Close()

    // Create concrete implementations
    orderRepo := pg.NewOrderRepository(db)
    emailSender := email.NewSMTPSender("smtp.gmail.com", "user", "pass")
    paymentProcessor := payment.NewStripeProcessor("sk_test_123")

    // Wire them together
    orderService := order.NewService(orderRepo, emailSender, paymentProcessor)

    // Use the service
    err := orderService.CreateOrder([]Item{{ProductID: 1, Quantity: 2}})
    if err != nil {
        log.Fatal(err)
    }
}
```

If you want to use a different email provider, you change one line here. No other file is affected.

## Event-Driven Decoupling

For even looser coupling, use events. Instead of calling dependencies directly, the service publishes events and other components react to them:

```go
type EventBus interface {
    Publish(event Event)
    Subscribe(eventType string, handler EventHandler)
}

type Event struct {
    Type    string
    Payload interface{}
}

type EventHandler func(event Event)

type OrderService struct {
    repo  OrderRepository
    event EventBus
}

func (s *OrderService) CreateOrder(items []Item) error {
    orderID, err := s.repo.Save(items)
    if err != nil {
        return err
    }

    // Publish event instead of calling dependencies directly
    s.event.Publish(Event{
        Type: "order.created",
        Payload: OrderCreatedPayload{OrderID: orderID, Items: items},
    })

    return nil
}
```

Now the email service and payment service subscribe to events independently:

```go
// Email service subscribes to order.created events
eventBus.Subscribe("order.created", func(e Event) {
    payload := e.Payload.(OrderCreatedPayload)
    emailSender.SendOrderConfirmation(payload.OrderID)
})

// Payment service subscribes to order.created events
eventBus.Subscribe("order.created", func(e Event) {
    payload := e.Payload.(OrderCreatedPayload)
    paymentProcessor.Charge(payload.OrderID, payload.Total)
})
```

The OrderService does not know about email or payment. It just publishes an event. This is maximum decoupling.

## Before and After Comparison

| Aspect | Tight Coupling | Loose Coupling |
|--------|---------------|----------------|
| **Testing** | Needs real database, email, payment | Mock everything easily |
| **Changing email provider** | Edit OrderService | Edit main() only |
| **Adding logging** | Edit every service | Add logging decorator |
| **Swapping database** | Edit every repository call | Implement new repository |
| **Code reuse** | Cannot reuse outside this project | Reuse interfaces anywhere |

Loose coupling is an investment. It costs more upfront to define interfaces and wire dependencies. But it saves enormous time and pain as the project grows. Every change becomes easier. Every test becomes simpler. Every component becomes reusable.
