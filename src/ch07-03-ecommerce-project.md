# Ecommerce Project Introduction

Alright, enough theory. Let us build something real.

Throughout the rest of this section, we are going to build a simple **ecommerce API**. This is not going to be a full Amazon clone, but it will cover all the fundamentals you need to build real backend applications.

## What We Will Build

Our ecommerce API will support these operations:

- **GET /products** - List all products
- **GET /products/{id}** - Get a single product by ID
- **POST /products** - Create a new product
- **PUT /products/{id}** - Update an existing product
- **DELETE /products/{id}** - Delete a product

These five operations cover the classic **CRUD** pattern: Create, Read, Update, Delete. Once you understand CRUD, you can build almost any backend application. Blogs, social media, task managers, they all follow the same pattern.

## Project Structure

Let me show you how we will organize the project. We are starting simple, and we will refactor as we go:

```
ecommerce-api/
  main.go           - Entry point, starts the server
  models/
    product.go      - Product model/struct
  handlers/
    product.go      - HTTP handlers for product routes
  go.mod            - Go module definition
  go.sum            - Dependency checksums
```

This is a flat, simple structure. Later, when we talk about project structure in depth, we will reorganize this into something more scalable. But for now, simple is good.

## Setting Up the Project

First, create the project directory and initialize a Go module:

```bash
mkdir ecommerce-api
cd ecommerce-api
go mod init github.com/yourname/ecommerce-api
```

The `go mod init` command creates a `go.mod` file that tracks your module name and dependencies. Think of it like `package.json` in Node.js.

## Creating the Product Model

Let us define what a Product looks like in our system:

```go
// models/product.go
package models

// Product represents an item in our ecommerce store
type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
    InStock     bool    `json:"in_stock"`
}
```

Those tags next to each field (like `` `json:"id"` ``) tell Go how to convert the struct to JSON and back. When we return a Product as JSON, the field names will match the tags instead of the Go field names. This is important because JSON convention uses snake_case, while Go uses PascalCase.

I remember being confused by these tags at first. "Why do I need them?" Well, without them, your JSON would have fields like `InStock` instead of `in_stock`. The tags give you control over the JSON format.

## Creating the Initial Server

Now let us write the main entry point. We will start with a basic server and a simple in-memory "database" (just a slice):

```go
// main.go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"

    "github.com/yourname/ecommerce-api/models"
)

// In-memory store (we will replace this with a real database later)
var products = []models.Product{
    {ID: 1, Name: "Go Programming Book", Description: "Learn Go from scratch", Price: 39.99, Category: "Books", InStock: true},
    {ID: 2, Name: "Mechanical Keyboard", Description: "Clicky and satisfying", Price: 129.99, Category: "Electronics", InStock: true},
    {ID: 3, Name: "Coffee Mug", Description: "For those late-night coding sessions", Price: 14.99, Category: "Accessories", InStock: false},
}

var nextID = 4

func productsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    switch r.Method {
    case http.MethodGet:
        json.NewEncoder(w).Encode(products)
    case http.MethodPost:
        var product models.Product
        err := json.NewDecoder(r.Body).Decode(&product)
        if err != nil {
            http.Error(w, "Invalid request body", http.StatusBadRequest)
            return
        }
        product.ID = nextID
        nextID++
        products = append(products, product)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(product)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/products", productsHandler)

    fmt.Println("Ecommerce API starting on port 8080...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Printf("Server error: %v\n", err)
    }
}
```

## What This Code Does

Let me walk through the key parts:

**In-memory store** - We use a `[]models.Product` slice to store products. This means our data will be lost when the server restarts. That is fine for now. We will add a real database later.

**`nextID`** - A simple counter to generate unique IDs for new products. In a real application, the database would handle this.

**`productsHandler`** - This one handler deals with both GET and POST requests on the `/products` path. We use a `switch` on `r.Method` to differentiate.

**`json.NewEncoder(w).Encode(products)`** - This converts our Go slice to JSON and writes it to the response. One line. That is the beauty of Go's standard library.

**`json.NewDecoder(r.Body).Decode(&product)`** - This reads JSON from the request body and converts it into a Go struct.

## Running the Project

Save the files and run:

```bash
go run main.go
```

Now you can test it:

```bash
# Get all products
curl http://localhost:8080/products

# Create a new product
curl -X POST http://localhost:8080/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Wireless Mouse", "description": "No more cables", "price": 29.99, "category": "Electronics", "in_stock": true}'
```

## What is Coming Next

Right now, everything is in `main.go`. It works, but it is already getting messy. Over the next few chapters, we will:

1. Handle GET requests properly with query and path parameters
2. Handle POST requests with validation
3. Deal with CORS and preflight requests
4. Refactor the code into a clean structure
5. Add proper routing with the chi router
6. Add middleware for logging, recovery, and authentication

This ecommerce project is our playground. We will keep building on it, breaking it, and improving it. That is how real development works. You start with something simple, and you iterate.

Let us move on to handling GET requests properly.
