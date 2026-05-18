# Clean Architecture

## What is Clean Architecture?

**Clean Architecture** was proposed by Robert C. Martin (Uncle Bob). The core idea is that the business logic of your application should not depend on frameworks, databases, or external services. Dependencies should point inward, from the outer layers to the inner layers.

The architecture has four concentric layers:

1. **Entities** (innermost) - Core business objects and rules
2. **Use Cases** - Application-specific business logic
3. **Interface Adapters** - Converters between use cases and external formats
4. **Frameworks & Drivers** (outermost) - Web frameworks, databases, UI

## The Dependency Rule

The most important rule: **dependencies point inward**. The inner layers do not know anything about the outer layers. Entities do not import use cases. Use cases do not import interface adapters. Interface adapters do not import frameworks.

This means:
- Your **entities** can be tested without a database
- Your **use cases** can be tested without a web framework
- You can swap PostgreSQL for MongoDB without touching the entities
- You can swap your HTTP framework without touching the business logic

## Clean Architecture in Go

Here is how to map the four layers to a Go project:

```
myapp/
├── domain/              # Entities layer
│   ├── user/
│   │   ├── user.go      # Entity
│   │   └── repo.go      # Repository interface
│   └── order/
│       ├── order.go     # Entity
│       └── repo.go      # Repository interface
├── usecase/             # Use Cases layer
│   ├── create_order.go  # Use case
│   └── get_user.go      # Use case
├── adapter/             # Interface Adapters layer
│   ├── handler/         # HTTP handlers
│   │   └── order.go
│   └── repository/      # Repository implementations
│       └── postgres/
│           └── order.go
└── main.go              # Frameworks & Drivers layer (wiring)
```

## Domain Layer (Entities)

The domain layer contains pure business logic. No imports from other layers:

```go
// domain/order/order.go
package order

import "errors"

type Status string

const (
    StatusPending   Status = "pending"
    StatusConfirmed Status = "confirmed"
    StatusCancelled Status = "cancelled"
)

type Order struct {
    ID     int64
    UserID int64
    Items  []Item
    Total  float64
    Status Status
}

type Item struct {
    ProductID int64
    Quantity  int
    Price     float64
}

// Business rules live here
var (
    ErrEmptyOrder      = errors.New("order must have at least one item")
    ErrInvalidQuantity = errors.New("quantity must be positive")
    ErrAlreadyCancelled = errors.New("order is already cancelled")
)

func NewOrder(userID int64) *Order {
    return &Order{
        UserID: userID,
        Status: StatusPending,
        Items:  []Item{},
    }
}

func (o *Order) AddItem(productID int64, quantity int, price float64) error {
    if quantity <= 0 {
        return ErrInvalidQuantity
    }

    // Check if item already exists
    for i, item := range o.Items {
        if item.ProductID == productID {
            o.Items[i].Quantity += quantity
            o.recalculate()
            return nil
        }
    }

    o.Items = append(o.Items, Item{
        ProductID: productID,
        Quantity:  quantity,
        Price:     price,
    })
    o.recalculate()
    return nil
}

func (o *Order) Cancel() error {
    if o.Status == StatusCancelled {
        return ErrAlreadyCancelled
    }
    o.Status = StatusCancelled
    return nil
}

func (o *Order) recalculate() {
    total := 0.0
    for _, item := range o.Items {
        total += item.Price * float64(item.Quantity)
    }
    o.Total = total
}
```

```go
// domain/order/repo.go
package order

type Repository interface {
    Save(order *Order) error
    FindByID(id int64) (*Order, error)
    FindByUserID(userID int64) ([]*Order, error)
    Update(order *Order) error
}
```

## Use Case Layer

Use cases implement application-specific business logic. They orchestrate entities and repositories:

```go
// usecase/create_order.go
package usecase

import (
    "fmt"
    "myapp/domain/order"
    "myapp/domain/product"
)

type CreateOrderInput struct {
    UserID int64
    Items  []struct {
        ProductID int64
        Quantity  int
    }
}

type CreateOrderOutput struct {
    OrderID int64
    Total   float64
    Status  order.Status
}

type CreateOrderUseCase struct {
    orderRepo   order.Repository
    productRepo product.Repository
}

func NewCreateOrderUseCase(
    orderRepo order.Repository,
    productRepo product.Repository,
) *CreateOrderUseCase {
    return &CreateOrderUseCase{
        orderRepo:   orderRepo,
        productRepo: productRepo,
    }
}

func (uc *CreateOrderUseCase) Execute(input CreateOrderInput) (*CreateOrderOutput, error) {
    // Create the order entity
    o := order.NewOrder(input.UserID)

    // Add items with product prices
    for _, item := range input.Items {
        p, err := uc.productRepo.FindByID(item.ProductID)
        if err != nil {
            return nil, fmt.Errorf("product %d not found: %w", item.ProductID, err)
        }
        if err := o.AddItem(p.ID, item.Quantity, p.Price); err != nil {
            return nil, err
        }
    }

    // Persist the order
    if err := uc.orderRepo.Save(o); err != nil {
        return nil, fmt.Errorf("failed to save order: %w", err)
    }

    return &CreateOrderOutput{
        OrderID: o.ID,
        Total:   o.Total,
        Status:  o.Status,
    }, nil
}
```

## Interface Adapter Layer

### HTTP Handler (Controller)

```go
// adapter/handler/order.go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    "myapp/usecase"
)

type OrderHandler struct {
    createOrderUC *usecase.CreateOrderUseCase
    getOrderUC    *usecase.GetOrderUseCase
}

func NewOrderHandler(
    createOrderUC *usecase.CreateOrderUseCase,
    getOrderUC *usecase.GetOrderUseCase,
) *OrderHandler {
    return &OrderHandler{
        createOrderUC: createOrderUC,
        getOrderUC:    getOrderUC,
    }
}

func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Items []struct {
            ProductID int64 `json:"product_id"`
            Quantity  int   `json:"quantity"`
        } `json:"items"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    userID := r.Context().Value("user_id").(int64)

    input := usecase.CreateOrderInput{
        UserID: userID,
        Items:  req.Items,
    }

    output, err := h.createOrderUC.Execute(input)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(output)
}
```

### Repository Implementation

```go
// adapter/repository/postgres/order.go
package postgres

import (
    "database/sql"
    "myapp/domain/order"
)

type OrderRepository struct {
    db *sql.DB
}

func NewOrderRepository(db *sql.DB) *OrderRepository {
    return &OrderRepository{db: db}
}

func (r *OrderRepository) Save(o *order.Order) error {
    // Database implementation
    // ...
    return nil
}

func (r *OrderRepository) FindByID(id int64) (*order.Order, error) {
    // Database implementation
    // ...
    return nil, nil
}
```

## Wiring in main()

```go
// main.go
package main

import (
    "database/sql"
    "net/http"

    _ "github.com/lib/pq"

    "myapp/adapter/handler"
    pg "myapp/adapter/repository/postgres"
    "myapp/domain/order"
    "myapp/domain/product"
    "myapp/usecase"
)

func main() {
    db, _ := sql.Open("postgres", "postgres://localhost/myapp?sslmode=disable")
    defer db.Close()

    // Repositories (interface adapters)
    var orderRepo order.Repository = pg.NewOrderRepository(db)
    var productRepo product.Repository = pg.NewProductRepository(db)

    // Use cases
    createOrderUC := usecase.NewCreateOrderUseCase(orderRepo, productRepo)
    getOrderUC := usecase.NewGetOrderUseCase(orderRepo)

    // Handlers (interface adapters)
    orderHandler := handler.NewOrderHandler(createOrderUC, getOrderUC)

    // Routes
    mux := http.NewServeMux()
    mux.HandleFunc("POST /api/orders", orderHandler.Create)
    mux.HandleFunc("GET /api/orders/{id}", orderHandler.Get)

    http.ListenAndServe(":8080", mux)
}
```

## Benefits of Clean Architecture

**Testable** - Each layer can be tested independently. Use cases are tested with mock repositories. Entities are tested with pure Go code.

**Maintainable** - Business logic is in one place. Changes to the database or web framework do not affect the core logic.

**Independent** - The domain layer has no external dependencies. You can change frameworks, databases, or message queues without touching the business rules.

**Flexible** - Adding a new use case does not affect existing ones. Adding a new handler (gRPC, CLI) uses the same use cases.

Clean Architecture takes more code to set up than putting everything in one file. But it pays off as the project grows. Every change is localized. Every test is isolated. Every layer has a clear responsibility.
