# Domain Driven Design (DDD)

## What is DDD?

**Domain Driven Design** is an approach to software development that organizes code around the **business domain**. Instead of thinking in terms of databases, APIs, and controllers, you think in terms of the business: customers, orders, products, shipments.

The idea comes from Eric Evans' book. The core principle is: **the business logic should be the center of your application, not a side effect of your database or framework**.

I used to start every project by designing database tables. Then I would write CRUD endpoints around those tables. The result? The database schema dictated my entire application. When the business rules changed, I had to change the database first, and everything downstream broke.

DDD flips this around. You start with the domain. The database is just a detail.

## Key Concepts

### Entities

An **entity** is an object with a distinct identity. Two users with the same name are still different users because they have different IDs. Entities have identity that persists over time.

```go
type User struct {
    ID       int64
    Email    string
    Name     string
    Password string
}
```

Even if you change the email and name, it is still the same user because the ID stays the same.

### Value Objects

A **value object** has no identity. It is defined by its attributes. Two money objects with $10 are the same. Two addresses with the same street, city, and zip are the same.

```go
type Money struct {
    Amount   float64
    Currency string
}

func (m Money) Equals(other Money) bool {
    return m.Amount == other.Amount && m.Currency == other.Currency
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, fmt.Errorf("cannot add different currencies")
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}
```

Value objects are immutable. You do not modify them. You create new ones.

### Aggregates

An **aggregate** is a cluster of entities and value objects that are treated as a single unit. Every aggregate has a **root entity**. All access to the aggregate goes through the root.

```go
// Order is the aggregate root
type Order struct {
    ID        int64
    UserID    int64
    Items     []OrderItem  // entities within the aggregate
    Total     Money        // value object
    Status    OrderStatus  // value object
}

// OrderItem is part of the Order aggregate
type OrderItem struct {
    ProductID int64
    Quantity  int
    Price     Money
}
```

You do not create or modify `OrderItem` directly. You go through the `Order`. This ensures business rules are always enforced.

### Repositories

A **repository** is the interface for saving and loading aggregates. It hides the storage details. The domain does not know if data is stored in PostgreSQL, MongoDB, or memory.

```go
type OrderRepository interface {
    FindByID(id int64) (*Order, error)
    FindByUserID(userID int64) ([]*Order, error)
    Save(order *Order) error
    Delete(id int64) error
}
```

### Domain Services

A **domain service** contains business logic that does not naturally belong to any entity or value object. It coordinates between multiple aggregates.

```go
type OrderService struct {
    orderRepo   OrderRepository
    productRepo ProductRepository
}

func (s *OrderService) PlaceOrder(userID int64, items []OrderItem) (*Order, error) {
    // Calculate total
    total := Money{Currency: "USD"}
    for _, item := range items {
        product, err := s.productRepo.FindByID(item.ProductID)
        if err != nil {
            return nil, fmt.Errorf("product not found: %w", err)
        }
        // Business rule: check stock
        if product.Stock < item.Quantity {
            return nil, fmt.Errorf("insufficient stock for product %d", item.ProductID)
        }
        total, _ = total.Add(item.Price)
    }

    // Create order
    order := &Order{
        UserID: userID,
        Items:  items,
        Total:  total,
        Status: OrderStatusPending,
    }

    // Save order
    if err := s.orderRepo.Save(order); err != nil {
        return nil, fmt.Errorf("failed to save order: %w", err)
    }

    return order, nil
}
```

### Domain Events

A **domain event** represents something that happened in the domain. Other parts of the system can react to these events:

```go
type DomainEvent interface {
    EventName() string
    OccurredAt() time.Time
}

type OrderPlacedEvent struct {
    OrderID   int64
    UserID    int64
    Total     Money
    Timestamp time.Time
}

func (e OrderPlacedEvent) EventName() string    { return "order.placed" }
func (e OrderPlacedEvent) OccurredAt() time.Time { return e.Timestamp }
```

When an order is placed, you might want to send a confirmation email, update inventory, and notify the warehouse. Events let you do this without coupling the order logic to email or inventory.

## Bounded Contexts

A **bounded context** is a boundary within which a particular domain model applies. The same concept can mean different things in different contexts.

For example, in an ecommerce system:
- In the **Order context**, a Product has a price and a name.
- In the **Inventory context**, a Product has a stock quantity and a warehouse location.
- In the **Shipping context**, a Product has a weight and dimensions.

Same product, different models for different contexts. Trying to make one model fit all contexts leads to a bloated, confusing mess.

## DDD in Go: Folder Structure

Here is how I organize a Go project using DDD:

```
ecommerce/
├── domain/
│   ├── order/
│   │   ├── order.go        # entity + value objects
│   │   ├── repository.go   # repository interface
│   │   └── service.go      # domain service
│   ├── product/
│   │   ├── product.go
│   │   └── repository.go
│   └── user/
│       ├── user.go
│       └── repository.go
├── application/
│   └── order/
│       └── service.go      # application service (use cases)
├── infrastructure/
│   ├── persistence/
│   │   ├── postgres/
│   │   │   ├── order_repo.go
│   │   │   └── product_repo.go
│   │   └── memory/
│   │       └── order_repo.go
│   └── notification/
│       └── email.go
└── presentation/
    └── http/
        └── handler/
            └── order_handler.go
```

The **domain** package has no external dependencies. It only depends on standard library types. The **infrastructure** package implements the interfaces defined in the domain. The **application** package coordinates use cases. The **presentation** package handles HTTP requests.

## Do You Need Full DDD?

Not every project needs full DDD. A simple CRUD app does not need aggregates, domain events, and bounded contexts. But the ideas are still valuable:

- **Think about the domain first**, not the database
- **Keep business logic separate** from infrastructure
- **Use repositories** to hide storage details
- **Model your types carefully** (entities vs value objects)

Start simple. Add DDD concepts as the project grows. When the business logic gets complex, you will be glad you have the structure.

## My Personal Note

I first heard about DDD and thought it was way too complex for my projects. All those terms: aggregates, bounded contexts, domain events. It sounded like enterprise over-engineering.

But then I built an ecommerce system the "simple" way. I put business logic in my HTTP handlers. I queried the database directly from my services. I had SQL strings scattered everywhere. When the business rules changed, I had to hunt through handlers to find the logic. It was a mess.

When I restructured the project with DDD principles, things clicked. The domain was pure Go code with no database imports. The business rules were in one place. Tests were easy because I could mock the repository. It was not more complex. It was simpler, because everything had its proper place.

DDD is not about adding complexity. It is about putting things where they belong.
