# Advanced Routing in Go

By now, you have probably noticed the limitations of Go's default `DefaultServeMux`. It works, but it is basic. You cannot match path parameters like `/products/{id}`. You cannot match HTTP methods easily. You cannot group routes or apply middleware to specific groups.

That is where **third-party routers** come in. Let me show you the options and then focus on my favorite: **chi**.

## The Problem with DefaultServeMux

Here are the main limitations:

**No path parameters** - You cannot define routes like `/users/{id}` or `/posts/{postId}/comments/{commentId}`. You have to parse the URL manually, which is tedious and error-prone.

**No method matching** - The default mux matches on path only. You have to check `r.Method` inside each handler with a switch statement.

**No route groups** - You cannot group routes under a common prefix with shared middleware. Everything is flat.

**No middleware support** - There is no built-in way to chain middleware with the default mux.

## Third-Party Router Options

There are several popular routers for Go:

| Router | Pros | Cons |
|--------|------|------|
| **gorilla/mux** | Feature-rich, widely used | No longer maintained, heavier |
| **chi** | Lightweight, stdlib compatible, well-maintained | Less features than gin |
| **gin** | Fast, lots of features, large community | Different API from stdlib, opinionated |
| **echo** | Simple, fast, good docs | Different API from stdlib |

I prefer **chi** because it is 100% compatible with Go's standard library. If you know how to use `net/http`, you already know most of chi. It adds just what you need without reinventing everything.

## Setting Up Chi

First, install chi:

```bash
go get github.com/go-chi/chi/v5
```

Notice we use `v5` which is the latest version. Now let us set up a basic chi router:

```go
package main

import (
    "encoding/json"
    "net/http"

    "github.com/go-chi/chi/v5"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

var products = []Product{
    {ID: 1, Name: "Go Book", Price: 39.99},
    {ID: 2, Name: "Keyboard", Price: 129.99},
}

func main() {
    r := chi.NewRouter()
    
    // Define routes with method matching!
    r.Get("/products", listProducts)
    r.Post("/products", createProduct)
    r.Get("/products/{id}", getProduct)
    r.Put("/products/{id}", updateProduct)
    r.Delete("/products/{id}", deleteProduct)
    
    http.ListenAndServe(":8080", r)
}

func listProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(products)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    var p Product
    json.NewDecoder(r.Body).Decode(&p)
    p.ID = len(products) + 1
    products = append(products, p)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(p)
}

func getProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    // Path parameters made easy!
    idStr := chi.URLParam(r, "id")
    // Convert and find...
    for _, p := range products {
        if strconv.Itoa(p.ID) == idStr {
            json.NewEncoder(w).Encode(p)
            return
        }
    }
    http.Error(w, "Not found", http.StatusNotFound)
}

func updateProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"message": "updated"})
}

func deleteProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusNoContent)
}
```

Look at that. `r.Get("/products/{id}", getProduct)` gives you method matching and path parameters in one clean line.

## Path Parameters

This is the feature I missed the most with the default mux. With chi, extracting path parameters is simple:

```go
func getProduct(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    fmt.Printf("Looking for product with ID: %s\n", id)
    
    // Convert to int
    productID, err := strconv.Atoi(id)
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    
    // Find the product...
}
```

No more manual string parsing. Chi handles the URL matching for you and extracts the parameters.

## Route Groups

Route groups let you organize related routes and apply middleware to them. This is super useful:

```go
func main() {
    r := chi.NewRouter()
    
    // Public routes (no auth needed)
    r.Group(func(r chi.Router) {
        r.Get("/products", listProducts)
        r.Get("/products/{id}", getProduct)
    })
    
    // Admin routes (auth required)
    r.Group(func(r chi.Router) {
        r.Use(authMiddleware)  // Only these routes need auth
        r.Post("/products", createProduct)
        r.Put("/products/{id}", updateProduct)
        r.Delete("/products/{id}", deleteProduct)
    })
    
    // API routes under /api prefix
    r.Route("/api", func(r chi.Router) {
        r.Get("/products", listProducts)
        r.Get("/products/{id}", getProduct)
    })
    
    http.ListenAndServe(":8080", r)
}
```

The `r.Group()` call creates a group of routes with shared middleware. The `r.Route()` call does the same but also adds a URL prefix. So `r.Route("/api", ...)` means all routes inside start with `/api`.

## Middleware with Chi

Chi makes middleware easy. You can apply middleware globally or to specific route groups:

```go
func main() {
    r := chi.NewRouter()
    
    // Global middleware (applies to all routes)
    r.Use(loggingMiddleware)
    r.Use(recoveryMiddleware)
    
    // Public routes
    r.Get("/", homeHandler)
    r.Get("/products", listProducts)
    
    // Protected routes with extra middleware
    r.Group(func(r chi.Router) {
        r.Use(authMiddleware)
        r.Post("/products", createProduct)
        r.Delete("/products/{id}", deleteProduct)
    })
    
    http.ListenAndServe(":8080", r)
}
```

Middleware runs in the order you add it. So in this example, every request goes through logging, then recovery. Protected routes also go through auth middleware.

## Built-in Chi Middleware

Chi comes with some useful middleware in the `chi/middleware` package:

```go
import "github.com/go-chi/chi/v5/middleware"

func main() {
    r := chi.NewRouter()
    
    // Useful built-in middleware
    r.Use(middleware.Logger)       // Log every request
    r.Use(middleware.Recoverer)    // Recover from panics
    r.Use(middleware.RequestID)    // Add unique request ID
    r.Use(middleware.Timeout(30 * time.Second))  // Timeout long requests
    r.Use(middleware.Compress(5))  // Gzip compression
    
    r.Get("/products", listProducts)
    
    http.ListenAndServe(":8080", r)
}
```

Just a few lines and your server has logging, panic recovery, request IDs, timeouts, and compression. That is production-ready stuff.

## Complete Example with Chi

Here is a complete, well-structured ecommerce API with chi:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
}

var products = []Product{
    {ID: 1, Name: "Go Book", Description: "Learn Go", Price: 39.99},
    {ID: 2, Name: "Keyboard", Description: "Mechanical", Price: 129.99},
}
var nextID = 3

func main() {
    r := chi.NewRouter()
    
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(30 * time.Second))
    
    r.Route("/products", func(r chi.Router) {
        r.Get("/", listProducts)
        r.Post("/", createProduct)
        r.Route("/{id}", func(r chi.Router) {
            r.Get("/", getProduct)
            r.Put("/", updateProduct)
            r.Delete("/", deleteProduct)
        })
    })
    
    fmt.Println("Server on :8080")
    http.ListenAndServe(":8080", r)
}

func listProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(products)
}

func getProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    id, err := strconv.Atoi(chi.URLParam(r, "id"))
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    for _, p := range products {
        if p.ID == id {
            json.NewEncoder(w).Encode(p)
            return
        }
    }
    http.Error(w, "Not found", http.StatusNotFound)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    var p Product
    if err := json.NewDecoder(r.Body).Decode(&p); err != nil {
        http.Error(w, "Invalid body", http.StatusBadRequest)
        return
    }
    p.ID = nextID
    nextID++
    products = append(products, p)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(p)
}

func updateProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte(`{"message": "update not implemented yet"}`))
}

func deleteProduct(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNoContent)
}
```

Switching from the default mux to chi is one of the best upgrades you can make to a Go web project. It gives you method matching, path parameters, route groups, and middleware support, all while staying compatible with the standard library. If you are building anything beyond a simple server, use chi.
