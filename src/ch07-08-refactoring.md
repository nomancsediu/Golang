# Refactoring Codebase

I have a confession. When I first started coding, I thought refactoring was something you do at the end, when everything is "done." I could not have been more wrong. Refactoring is something you do constantly, as you build. Because if you do not, your code becomes a mess faster than you think.

Let me show you what I mean.

## The Problem: Everything in main.go

By now, our ecommerce API has routes, handlers, models, and in-memory storage. And it is probably all crammed into one file. Something like this:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
}

var products = []Product{
    {ID: 1, Name: "Go Book", Description: "Learn Go", Price: 39.99, Category: "Books"},
}

var nextID = 2

func productsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    switch r.Method {
    case http.MethodGet:
        // Handle query params, filter, etc.
        category := r.URL.Query().Get("category")
        if category != "" {
            var filtered []Product
            for _, p := range products {
                if p.Category == category {
                    filtered = append(filtered, p)
                }
            }
            json.NewEncoder(w).Encode(filtered)
            return
        }
        json.NewEncoder(w).Encode(products)
    case http.MethodPost:
        var p Product
        if err := json.NewDecoder(r.Body).Decode(&p); err != nil {
            http.Error(w, "Invalid body", http.StatusBadRequest)
            return
        }
        if p.Name == "" {
            http.Error(w, "Name required", http.StatusBadRequest)
            return
        }
        p.ID = nextID
        nextID++
        products = append(products, p)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(p)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

// ... more handlers, more logic, everything in one file

func main() {
    http.HandleFunc("/products", productsHandler)
    fmt.Println("Server on :8080")
    http.ListenAndServe(":8080", nil)
}
```

This works. But as your API grows, this file becomes 300, 500, 1000 lines. Every time you add a new feature, you have to scroll through the entire file. It becomes hard to find things, hard to test things, and hard to change things without breaking something else.

## Separating Concerns

The key principle behind refactoring is **separation of concerns**. Each piece of code should do one thing and do it well. For a web API, we typically split code into these layers:

- **Models** - Data structures (Product, User, Order)
- **Handlers** - Handle HTTP requests and responses
- **Services** - Business logic (validation, calculations, rules)
- **Repository** - Database operations (CRUD)

Let us refactor step by step.

## Step 1: Move Models to Their Own Package

Create `models/product.go`:

```go
package models

// Product represents an item in our store
type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
}

// Validate checks if a product has all required fields
func (p *Product) Validate() map[string]string {
    errors := make(map[string]string)
    if p.Name == "" {
        errors["name"] = "Name is required"
    }
    if p.Price <= 0 {
        errors["price"] = "Price must be greater than zero"
    }
    if p.Category == "" {
        errors["category"] = "Category is required"
    }
    return errors
}
```

Notice how we moved the `Validate` method onto the struct. The model now owns its own validation logic. This is cleaner than having it scattered in the handler.

## Step 2: Create a Repository

The repository layer handles data storage. For now, it is in-memory. Later, we can swap it for a database without changing anything else.

Create `repository/product.go`:

```go
package repository

import "github.com/yourname/ecommerce-api/models"

// ProductRepository handles data storage for products
type ProductRepository struct {
    products []models.Product
    nextID   int
}

// NewProductRepository creates a new repository
func NewProductRepository() *ProductRepository {
    return &ProductRepository{
        products: []models.Product{
            {ID: 1, Name: "Go Book", Description: "Learn Go", Price: 39.99, Category: "Books"},
            {ID: 2, Name: "Keyboard", Description: "Mechanical", Price: 129.99, Category: "Electronics"},
        },
        nextID: 3,
    }
}

// GetAll returns all products
func (r *ProductRepository) GetAll() []models.Product {
    return r.products
}

// GetByCategory returns products filtered by category
func (r *ProductRepository) GetByCategory(category string) []models.Product {
    var filtered []models.Product
    for _, p := range r.products {
        if p.Category == category {
            filtered = append(filtered, p)
        }
    }
    return filtered
}

// GetByID returns a single product by ID
func (r *ProductRepository) GetByID(id int) (*models.Product, bool) {
    for _, p := range r.products {
        if p.ID == id {
            return &p, true
        }
    }
    return nil, false
}

// Create adds a new product and returns it with the assigned ID
func (r *ProductRepository) Create(product models.Product) models.Product {
    product.ID = r.nextID
    r.nextID++
    r.products = append(r.products, product)
    return product
}
```

Now all data operations are in one place. If we want to add a database later, we only change this file.

## Step 3: Create a Handler Struct with Dependencies

Create `handlers/product.go`:

```go
package handlers

import (
    "encoding/json"
    "net/http"

    "github.com/yourname/ecommerce-api/models"
    "github.com/yourname/ecommerce-api/repository"
)

// ProductHandler handles HTTP requests for products
type ProductHandler struct {
    repo *repository.ProductRepository
}

// NewProductHandler creates a new handler with dependencies
func NewProductHandler(repo *repository.ProductRepository) *ProductHandler {
    return &ProductHandler{repo: repo}
}

// GetAll handles GET /products
func (h *ProductHandler) GetAll(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    category := r.URL.Query().Get("category")
    
    var products []models.Product
    if category != "" {
        products = h.repo.GetByCategory(category)
    } else {
        products = h.repo.GetAll()
    }
    
    json.NewEncoder(w).Encode(products)
}

// Create handles POST /products
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    var product models.Product
    if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
        http.Error(w, `{"error": "Invalid JSON"}`, http.StatusBadRequest)
        return
    }
    
    if errors := product.Validate(); len(errors) > 0 {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]interface{}{"errors": errors})
        return
    }
    
    created := h.repo.Create(product)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(created)
}
```

Notice how the handler now has a **reference to the repository**. This is called **dependency injection**. Instead of accessing global variables, the handler receives its dependencies through the constructor.

## Step 4: Clean main.go

Now our main.go is clean and simple:

```go
package main

import (
    "fmt"
    "net/http"

    "github.com/yourname/ecommerce-api/handlers"
    "github.com/yourname/ecommerce-api/repository"
)

func main() {
    // Create dependencies
    productRepo := repository.NewProductRepository()
    productHandler := handlers.NewProductHandler(productRepo)
    
    // Register routes
    http.HandleFunc("/products", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            productHandler.GetAll(w, r)
        case http.MethodPost:
            productHandler.Create(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    })
    
    fmt.Println("Ecommerce API on :8080")
    http.ListenAndServe(":8080", nil)
}
```

Look at that. `main.go` is now just wiring things together. It creates the repository, passes it to the handler, and sets up routes. No business logic, no data access, no validation.

## Why This Matters

You might be thinking, "That is a lot more files for the same functionality." And you are right, in terms of line count. But here is what you gain:

**Testability** - You can test the repository without HTTP. You can test the handler with a mock repository. Each piece is testable in isolation.

**Maintainability** - When you need to add a database, you only change the repository. When you need to change validation, you only change the model. Changes are isolated.

**Readability** - Each file has one job. When you need to find how products are stored, you go to `repository/product.go`. When you need to find how requests are handled, you go to `handlers/product.go`.

**Scalability** - As the project grows, this structure scales. You add new handlers, new repositories, new models, and they all fit into the same pattern.

Refactoring is not about making code perfect on the first try. It is about continuously improving the structure as you learn more about what the code needs to do. Start simple, then refactor when things get messy. That is the real workflow.
