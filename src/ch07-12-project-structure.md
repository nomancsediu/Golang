# Project Structure in Go

One of the things that confused me most when learning Go was how to organize a project. In Node.js, there is a common pattern: `src/`, `routes/`, `controllers/`, `models/`. In Python, Django has its own conventions. But Go? Everyone seems to do it differently.

After building several Go projects, I have settled on a structure that works well for web APIs. Let me share it with you.

## Standard Go Project Layout

The Go community has a semi-official project layout. Not everyone follows it, but it is widely recognized. Here are the main directories:

**`cmd/`** - Contains the entry points of your application. Each subdirectory represents a different executable. For a web API, you might have `cmd/api/` for the API server and `cmd/migrate/` for database migrations.

**`internal/`** - Private code that cannot be imported by other projects. Go enforces this at the compiler level. Put your business logic, handlers, and services here.

**`pkg/`** - Public code that could be imported by other projects. If you are building a library alongside your API, its public packages go here. Many projects skip this directory entirely if they are not building reusable packages.

**`go.mod`** and **`go.sum`** - Module definition and dependency checksums. These are like `package.json` and `package-lock.json` in Node.js.

## Organizing by Feature vs by Layer

There are two main ways to organize Go code:

**By layer:**
```
internal/
  handlers/     - All HTTP handlers
  services/     - All business logic
  repository/   - All database access
  models/       - All data structures
```

**By feature:**
```
internal/
  products/
    handler.go
    service.go
    repository.go
    model.go
  users/
    handler.go
    service.go
    repository.go
    model.go
  orders/
    handler.go
    service.go
    repository.go
    model.go
```

For small to medium projects, organizing by layer is simpler and easier to understand. For large projects with many features, organizing by feature keeps related code together and reduces the mental overhead of switching between directories.

I prefer a **hybrid approach**: organize models by feature, but keep handlers and services together. Here is what that looks like:

```
ecommerce-api/
  cmd/
    api/
      main.go          - Application entry point
  internal/
    handler/
      product.go       - Product HTTP handlers
      user.go          - User HTTP handlers
      order.go         - Order HTTP handlers
      response.go      - Helper functions for responses
    service/
      product.go       - Product business logic
      user.go          - User business logic
      order.go         - Order business logic
    repository/
      product.go       - Product database operations
      user.go          - User database operations
      order.go         - Order database operations
    model/
      product.go       - Product struct and validation
      user.go          - User struct and validation
      order.go         - Order struct and validation
    middleware/
      auth.go          - Authentication middleware
      cors.go          - CORS middleware
      logging.go       - Logging middleware
      recovery.go      - Panic recovery middleware
    router/
      router.go        - Route definitions
  pkg/
    database/
      postgres.go      - Database connection setup
    config/
      config.go        - Configuration loading
  go.mod
  go.sum
  .env                  - Environment variables
```

## The Entry Point: cmd/api/main.go

Your `main.go` should be minimal. It just wires everything together:

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/yourname/ecommerce-api/internal/handler"
    "github.com/yourname/ecommerce-api/internal/middleware"
    "github.com/yourname/ecommerce-api/internal/repository"
    "github.com/yourname/ecommerce-api/internal/router"
    "github.com/yourname/ecommerce-api/internal/service"
    "github.com/yourname/ecommerce-api/pkg/config"
    "github.com/yourname/ecommerce-api/pkg/database"
)

func main() {
    // Load configuration
    cfg := config.Load()
    
    // Connect to database
    db, err := database.Connect(cfg.DatabaseURL)
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer db.Close()
    
    // Create repositories (data access layer)
    productRepo := repository.NewProductRepository(db)
    userRepo := repository.NewUserRepository(db)
    
    // Create services (business logic layer)
    productService := service.NewProductService(productRepo)
    userService := service.NewUserService(userRepo)
    
    // Create handlers (HTTP layer)
    productHandler := handler.NewProductHandler(productService)
    userHandler := handler.NewUserHandler(userService)
    
    // Set up middleware
    middlewares := []func(http.Handler) http.Handler{
        middleware.Recovery(),
        middleware.Logging(),
        middleware.CORS(),
    }
    
    // Set up router
    r := router.New(productHandler, userHandler, middlewares)
    
    // Start the server
    addr := fmt.Sprintf(":%s", cfg.Port)
    fmt.Printf("Server starting on %s\n", addr)
    if err := http.ListenAndServe(addr, r); err != nil {
        log.Fatalf("Server failed: %v", err)
    }
}
```

See how the flow goes: config -> database -> repositories -> services -> handlers -> router -> server. Each layer depends on the previous one. This is **dependency injection** in action.

## The Router: internal/router/router.go

Keep all your route definitions in one place:

```go
package router

import (
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    "github.com/yourname/ecommerce-api/internal/handler"
    "github.com/yourname/ecommerce-api/internal/middleware"
)

func New(
    productHandler *handler.ProductHandler,
    userHandler *handler.UserHandler,
    customMiddleware []func(http.Handler) http.Handler,
) http.Handler {
    r := chi.NewRouter()
    
    // Built-in chi middleware
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    
    // Custom middleware
    for _, m := range customMiddleware {
        r.Use(m)
    }
    
    // Health check
    r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    // API routes
    r.Route("/api/v1", func(r chi.Router) {
        // Public routes
        r.Route("/products", func(r chi.Router) {
            r.Get("/", productHandler.List)
            r.Get("/{id}", productHandler.Get)
        })
        
        r.Route("/auth", func(r chi.Router) {
            r.Post("/login", userHandler.Login)
            r.Post("/register", userHandler.Register)
        })
        
        // Protected routes
        r.Group(func(r chi.Router) {
            r.Use(middleware.Auth)
            
            r.Route("/products", func(r chi.Router) {
                r.Post("/", productHandler.Create)
                r.Put("/{id}", productHandler.Update)
                r.Delete("/{id}", productHandler.Delete)
            })
        })
    })
    
    return r
}
```

## The Handler Layer: internal/handler/product.go

Handlers deal with HTTP concerns only. They parse requests, call services, and format responses:

```go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"

    "github.com/go-chi/chi/v5"
    "github.com/yourname/ecommerce-api/internal/model"
    "github.com/yourname/ecommerce-api/internal/service"
)

type ProductHandler struct {
    service *service.ProductService
}

func NewProductHandler(s *service.ProductService) *ProductHandler {
    return &ProductHandler{service: s}
}

func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    products, err := h.service.GetAll(r.Context())
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to list products")
        return
    }
    respondJSON(w, http.StatusOK, products)
}

func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(chi.URLParam(r, "id"))
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid product ID")
        return
    }
    
    product, err := h.service.GetByID(r.Context(), id)
    if err != nil {
        respondError(w, http.StatusNotFound, "Product not found")
        return
    }
    respondJSON(w, http.StatusOK, product)
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var input model.CreateProductInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    if errors := input.Validate(); len(errors) > 0 {
        respondJSON(w, http.StatusBadRequest, map[string]interface{}{"errors": errors})
        return
    }
    
    product, err := h.service.Create(r.Context(), input)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to create product")
        return
    }
    respondJSON(w, http.StatusCreated, product)
}

// Helper functions
func respondJSON(w http.ResponseWriter, code int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, code int, message string) {
    respondJSON(w, code, map[string]string{"error": message})
}
```

## The Service Layer: internal/service/product.go

Services contain business logic. They do not know about HTTP:

```go
package service

import (
    "context"

    "github.com/yourname/ecommerce-api/internal/model"
    "github.com/yourname/ecommerce-api/internal/repository"
)

type ProductService struct {
    repo *repository.ProductRepository
}

func NewProductService(repo *repository.ProductRepository) *ProductService {
    return &ProductService{repo: repo}
}

func (s *ProductService) GetAll(ctx context.Context) ([]model.Product, error) {
    return s.repo.GetAll(ctx)
}

func (s *ProductService) GetByID(ctx context.Context, id int) (*model.Product, error) {
    return s.repo.GetByID(ctx, id)
}

func (s *ProductService) Create(ctx context.Context, input model.CreateProductInput) (*model.Product, error) {
    product := model.Product{
        Name:        input.Name,
        Description: input.Description,
        Price:       input.Price,
        Category:    input.Category,
    }
    return s.repo.Create(ctx, product)
}
```

## Configuration: pkg/config/config.go

Never hardcode values. Use configuration:

```go
package config

import "os"

type Config struct {
    Port        string
    DatabaseURL string
    JWTSecret   string
    Environment string
}

func Load() *Config {
    return &Config{
        Port:        getEnv("PORT", "8080"),
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/ecommerce"),
        JWTSecret:   getEnv("JWT_SECRET", "change-me-in-production"),
        Environment: getEnv("ENV", "development"),
    }
}

func getEnv(key, fallback string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return fallback
}
```

## go.mod and go.sum

Your `go.mod` file tracks your module and its dependencies:

```
module github.com/yourname/ecommerce-api

go 1.21

require (
    github.com/go-chi/chi/v5 v5.0.10
    github.com/lib/pq v1.10.9
)
```

When you run `go get` to install a package, Go updates both `go.mod` and `go.sum` automatically. Always commit both files to version control.

## Key Takeaways

1. **Use `cmd/` for entry points** - Keep `main.go` thin
2. **Use `internal/` for private code** - Go enforces this boundary
3. **Separate by layer** - Handlers, services, repositories, models
4. **Use dependency injection** - Pass dependencies through constructors
5. **Keep handlers dumb** - They should only parse requests and format responses
6. **Keep services HTTP-free** - Business logic should not know about HTTP
7. **Use configuration** - Never hardcode values
8. **Version your API** - Use `/api/v1/` prefix for routes

This structure is not the only way to organize a Go project, but it is one that has worked well for me and many others. Start with this, and adapt it as your project grows. The most important thing is consistency. Once you pick a structure, stick with it across your entire project.
