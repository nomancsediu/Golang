# GET Request in Go

The **GET request** is the most common type of HTTP request. When you type a URL into your browser and hit Enter, that is a GET request. When you click a link, that is a GET request. When an app fetches data from an API, that is usually a GET request too.

GET requests are for **reading data**. They should not change anything on the server. No creating, updating, or deleting. Just reading.

Let us see how to handle GET requests properly in Go.

## Basic GET Handler

Here is a simple handler that returns a list of products:

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(products)
}
```

That is it. Three lines of code to return JSON data. The `json.NewEncoder(w).Encode(products)` call does two things: it converts our Go data to JSON, and it writes that JSON directly to the response writer.

## Setting Headers

The line `w.Header().Set("Content-Type", "application/json")` is important. It tells the client that the response body contains JSON data. Without this header, the client might not know how to parse the response.

You can set other headers too:

```go
w.Header().Set("Content-Type", "application/json")
w.Header().Set("X-Request-ID", "abc-123")
w.Header().Set("Cache-Control", "no-cache")
```

**Important:** You must set headers BEFORE you write the response body or call `w.WriteHeader()`. Once the response starts being sent, you cannot change the headers anymore.

## Reading Query Parameters

Query parameters are the key-value pairs that come after the `?` in a URL. For example, in `/products?category=books&sort=price`, the query parameters are `category=books` and `sort=price`.

In Go, you read them with `r.URL.Query()`:

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    query := r.URL.Query()
    category := query.Get("category")
    
    // If category is specified, filter products
    if category != "" {
        var filtered []models.Product
        for _, p := range products {
            if p.Category == category {
                filtered = append(filtered, p)
            }
        }
        json.NewEncoder(w).Encode(filtered)
        return
    }
    
    // No filter, return all products
    json.NewEncoder(w).Encode(products)
}
```

Now you can do:
- `GET /products` returns all products
- `GET /products?category=Books` returns only books

The `query.Get("category")` call returns an empty string if the parameter is not present. That is why we check `if category != ""`.

For parameters that can have multiple values (like `?id=1&id=2`), use `query["id"]` which returns a slice of strings.

## Reading Path Parameters

Path parameters are values embedded in the URL path itself, like `/products/3` where `3` is the product ID.

The bad news: Go's `net/http` default mux does not support path parameters. You cannot register a route like `/products/{id}` with the standard library.

The good news: There are easy workarounds, and we will use a proper router soon.

For now, here is a manual approach:

```go
func productHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    // Extract ID from the URL path manually
    // Path looks like /products/3
    path := strings.TrimPrefix(r.URL.Path, "/products/")
    
    if path == "" {
        // No ID, list all products
        json.NewEncoder(w).Encode(products)
        return
    }
    
    id, err := strconv.Atoi(path)
    if err != nil {
        http.Error(w, "Invalid product ID", http.StatusBadRequest)
        return
    }
    
    // Find the product
    for _, p := range products {
        if p.ID == id {
            json.NewEncoder(w).Encode(p)
            return
        }
    }
    
    // Product not found
    http.Error(w, "Product not found", http.StatusNotFound)
}
```

This works, but it is ugly. Parsing paths manually is error-prone and tedious. That is why we will switch to the **chi** router later, which makes this much cleaner:

```go
// With chi, path parameters are easy
r.Get("/products/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    // ...
})
```

## HTTP Status Codes

Every HTTP response includes a **status code** that tells the client what happened. Here are the ones you will use most often with GET requests:

| Code | Name | When to use |
|------|------|-------------|
| 200 | OK | Request succeeded, here is your data |
| 400 | Bad Request | The client sent something invalid |
| 404 | Not Found | The resource does not exist |
| 500 | Internal Server Error | Something went wrong on our side |

In Go, you set the status code with `w.WriteHeader()`:

```go
w.WriteHeader(http.StatusOK)     // 200
w.WriteHeader(http.StatusNotFound)  // 404
```

**Important note:** If you do not call `w.WriteHeader()`, Go will automatically send 200 OK when you start writing the response body. That is why our earlier examples did not need to explicitly set 200.

## Complete GET Example

Let us put it all together into a complete example:

```go
package main

import (
    "encoding/json"
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
    {ID: 2, Name: "Keyboard", Description: "Mechanical", Price: 129.99, Category: "Electronics"},
    {ID: 3, Name: "Mug", Description: "For coffee", Price: 14.99, Category: "Accessories"},
}

func productsHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    
    // Check for query parameters
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
}

func productDetailHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    
    path := strings.TrimPrefix(r.URL.Path, "/products/")
    id, err := strconv.Atoi(path)
    if err != nil {
        http.Error(w, `{"error": "Invalid product ID"}`, http.StatusBadRequest)
        return
    }
    
    for _, p := range products {
        if p.ID == id {
            json.NewEncoder(w).Encode(p)
            return
        }
    }
    
    http.Error(w, `{"error": "Product not found"}`, http.StatusNotFound)
}

func main() {
    http.HandleFunc("/products", productsHandler)
    http.HandleFunc("/products/", productDetailHandler)
    http.ListenAndServe(":8080", nil)
}
```

Notice that we register `/products/` (with trailing slash) for the detail handler. In Go's default mux, a path with a trailing slash matches any path that starts with it. So `/products/` will match `/products/1`, `/products/2`, etc.

GET requests are the foundation of any API. Once you can read query parameters, handle path parameters, and return proper status codes, you have the building blocks for reading any data.
