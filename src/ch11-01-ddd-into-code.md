# DDD Into Code

## Implementing DDD in a Real Go Project

In the previous section, we learned the concepts of Domain Driven Design: entities, value objects, aggregates, repositories, services, and bounded contexts. Now it is time to turn those concepts into actual code.

This is where I used to get stuck. The concepts made sense in theory, but how do I actually organize the files? How do the layers connect? Where does the database query go? Let me walk through a complete example.

## The Layered Architecture

DDD applications typically have four layers:

**Domain Layer** - The core business logic. Entities, value objects, domain errors. This layer has no dependencies on external packages.

**Application Layer** - Use cases and orchestration. It calls the domain layer and coordinates between repositories and services.

**Infrastructure Layer** - Technical implementations. Database repositories, external API clients, email senders.

**Presentation Layer** - How the outside world interacts with the application. HTTP handlers, gRPC servers, CLI commands.

The key rule: **dependencies point inward**. The domain layer depends on nothing. The application layer depends on the domain. The infrastructure depends on the domain. The presentation depends on the application.

## Folder Structure

Here is the folder structure for our ecommerce application:

```
ecommerce/
├── cmd/
│   └── server/
│       └── main.go
├── domain/
│   ├── order/
│   │   ├── order.go
│   │   ├── order_item.go
│   │   ├── repository.go
│   │   └── service.go
│   ├── product/
│   │   ├── product.go
│   │   └── repository.go
│   └── user/
│       ├── user.go
│       └── repository.go
├── application/
│   └── order/
│       └── service.go
├── infrastructure/
│   ├── persistence/
│   │   └── postgres/
│   │       ├── order_repository.go
│   │       ├── product_repository.go
│   │       └── user_repository.go
│   └── notification/
│       └── email_service.go
└── presentation/
    └── http/
        ├── handler/
        │   └── order_handler.go
        └── middleware/
            └── auth.go
```

## Domain Layer

The domain layer is the heart of the application. It contains pure Go types and business logic with no external dependencies:

```go
// domain/order/order.go
package order

import (
    "fmt"
    "time"

    "ecommerce/domain/product"
)

type Status string

const (
    StatusPending   Status = "pending"
    StatusConfirmed Status = "confirmed"
    StatusShipped   Status = "shipped"
    StatusDelivered Status = "delivered"
    StatusCancelled Status = "cancelled"
)

// Order is the aggregate root
type Order struct {
    ID        int64
    UserID    int64
    Items     []OrderItem
    Total     float64
    Status    Status
    CreatedAt time.Time
    UpdatedAt *time.Time
}

// Domain error
type OrderError struct {
    Code    string
    Message string
}

func (e OrderError) Error() string {
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

// Business logic methods on the entity
func (o *Order) AddItem(p product.Product, quantity int) error {
    // Business rule: cannot add items to a confirmed order
    if o.Status != StatusPending {
        return OrderError{Code: "ORDER_NOT_PENDING", Message: "cannot modify a non-pending order"}
    }

    // Business rule: quantity must be positive
    if quantity <= 0 {
        return OrderError{Code: "INVALID_QUANTITY", Message: "quantity must be positive"}
    }

    // Business rule: check if product is in stock
    if p.Stock < quantity {
        return OrderError{Code: "INSUFFICIENT_STOCK",
            Message: fmt.Sprintf("product %s has only %d in stock", p.Name, p.Stock)}
    }

    // Check if item already exists in the order
    for i, item := range o.Items {
        if item.ProductID == p.ID {
            o.Items[i].Quantity += quantity
            o.recalculateTotal()
            return nil
        }
    }

    // Add new item
    o.Items = append(o.Items, OrderItem{
        ProductID: p.ID,
        Name:      p.Name,
        Quantity:  quantity,
        Price:     p.Price,
    })
    o.recalculateTotal()
    return nil
}

func (o *Order) Cancel() error {
    if o.Status == StatusShipped || o.Status == StatusDelivered {
        return OrderError{Code: "CANNOT_CANCEL",
            Message: "cannot cancel a shipped or delivered order"}
    }
    o.Status = StatusCancelled
    return nil
}

func (o *Order) recalculateTotal() {
    total := 0.0
    for _, item := range o.Items {
        total += item.Price * float64(item.Quantity)
    }
    o.Total = total
}
```

```go
// domain/order/order_item.go
package order

type OrderItem struct {
    ID        int64
    OrderID   int64
    ProductID int64
    Name      string
    Quantity  int
    Price     float64
}
```

```go
// domain/order/repository.go
package order

// Repository interface defined in the domain layer
type Repository interface {
    Save(order *Order) error
    FindByID(id int64) (*Order, error)
    FindByUserID(userID int64) ([]*Order, error)
    Update(order *Order) error
    Delete(id int64) error
}
```

## Application Layer

The application layer contains **use cases**. It orchestrates domain objects and repositories:

```go
// application/order/service.go
package orderservice

import (
    "fmt"

    "ecommerce/domain/order"
    "ecommerce/domain/product"
)

type Service struct {
    orderRepo   order.Repository
    productRepo product.Repository
}

func NewService(orderRepo order.Repository, productRepo product.Repository) *Service {
    return &Service{
        orderRepo:   orderRepo,
        productRepo: productRepo,
    }
}

func (s *Service) CreateOrder(userID int64) (*order.Order, error) {
    o := &order.Order{
        UserID: userID,
        Status: order.StatusPending,
        Items:  []order.OrderItem{},
    }

    if err := s.orderRepo.Save(o); err != nil {
        return nil, fmt.Errorf("failed to create order: %w", err)
    }

    return o, nil
}

func (s *Service) AddItem(orderID, productID int64, quantity int) (*order.Order, error) {
    o, err := s.orderRepo.FindByID(orderID)
    if err != nil {
        return nil, fmt.Errorf("order not found: %w", err)
    }

    p, err := s.productRepo.FindByID(productID)
    if err != nil {
        return nil, fmt.Errorf("product not found: %w", err)
    }

    if err := o.AddItem(*p, quantity); err != nil {
        return nil, err
    }

    if err := s.orderRepo.Update(o); err != nil {
        return nil, fmt.Errorf("failed to update order: %w", err)
    }

    return o, nil
}

func (s *Service) GetOrder(id int64) (*order.Order, error) {
    return s.orderRepo.FindByID(id)
}

func (s *Service) CancelOrder(id int64) error {
    o, err := s.orderRepo.FindByID(id)
    if err != nil {
        return fmt.Errorf("order not found: %w", err)
    }

    if err := o.Cancel(); err != nil {
        return err
    }

    return s.orderRepo.Update(o)
}
```

## Infrastructure Layer

The infrastructure layer implements the domain interfaces:

```go
// infrastructure/persistence/postgres/order_repository.go
package postgres

import (
    "database/sql"
    "fmt"

    "ecommerce/domain/order"
)

type OrderRepository struct {
    db *sql.DB
}

func NewOrderRepository(db *sql.DB) *OrderRepository {
    return &OrderRepository{db: db}
}

func (r *OrderRepository) Save(o *order.Order) error {
    query := `
        INSERT INTO orders (user_id, status, total)
        VALUES ($1, $2, $3)
        RETURNING id, created_at`

    err := r.db.QueryRow(query, o.UserID, o.Status, o.Total).
        Scan(&o.ID, &o.CreatedAt)
    if err != nil {
        return fmt.Errorf("failed to save order: %w", err)
    }

    // Save order items
    for i := range o.Items {
        o.Items[i].OrderID = o.ID
        err := r.db.QueryRow(
            `INSERT INTO order_items (order_id, product_id, quantity, price)
             VALUES ($1, $2, $3, $4) RETURNING id`,
            o.Items[i].OrderID, o.Items[i].ProductID,
            o.Items[i].Quantity, o.Items[i].Price,
        ).Scan(&o.Items[i].ID)
        if err != nil {
            return fmt.Errorf("failed to save order item: %w", err)
        }
    }

    return nil
}

func (r *OrderRepository) FindByID(id int64) (*order.Order, error) {
    // Implementation details omitted for brevity
    // Query the orders table and order_items table
    // Build the Order aggregate from the results
    return nil, nil
}
```

## Presentation Layer

The presentation layer handles HTTP requests:

```go
// presentation/http/handler/order_handler.go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"

    "ecommerce/application/order"
)

type OrderHandler struct {
    service *orderservice.Service
}

func NewOrderHandler(service *orderservice.Service) *OrderHandler {
    return &OrderHandler{service: service}
}

func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Get user ID from auth context
    userID, _ := r.Context().Value("user_id").(int64)

    order, err := h.service.CreateOrder(userID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}

func (h *OrderHandler) AddItem(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ProductID int64 `json:"product_id"`
        Quantity  int   `json:"quantity"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    orderID, _ := strconv.ParseInt(r.PathValue("id"), 10, 64)
    order, err := h.service.AddItem(orderID, req.ProductID, req.Quantity)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(order)
}
```

## Wiring It Together in main()

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"

    _ "github.com/lib/pq"

    orderservice "ecommerce/application/order"
    pg "ecommerce/infrastructure/persistence/postgres"
    "ecommerce/presentation/http/handler"
)

func main() {
    db, _ := sql.Open("postgres", "postgres://localhost/myapp?sslmode=disable")
    defer db.Close()

    // Infrastructure layer
    orderRepo := pg.NewOrderRepository(db)
    productRepo := pg.NewProductRepository(db)

    // Application layer
    orderService := orderservice.NewService(orderRepo, productRepo)

    // Presentation layer
    orderHandler := handler.NewOrderHandler(orderService)

    // Routes
    mux := http.NewServeMux()
    mux.HandleFunc("POST /api/orders", orderHandler.Create)
    mux.HandleFunc("POST /api/orders/{id}/items", orderHandler.AddItem)

    log.Println("Server running on :8080")
    http.ListenAndServe(":8080", mux)
}
```

The **dependency direction** flows inward. main() creates concrete implementations and injects them. The handlers only know about the service interface. The service only knows about the repository interface. The domain knows nothing about any of this.

This is DDD in practice. The domain is pure. The infrastructure is a detail. The application coordinates. The presentation delivers.
