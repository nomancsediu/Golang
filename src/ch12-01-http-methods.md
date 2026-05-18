# HTTP Methods

## The Verbs of the Web

HTTP methods tell the server **what kind of action** the client wants to perform. They are the verbs in the HTTP protocol. The URL is the noun (the resource), and the method is the verb (the action).

Understanding HTTP methods is essential for building RESTful APIs. Each method has a specific meaning, and using them correctly makes your API intuitive and predictable.

## Common HTTP Methods

### GET - Read

**GET** retrieves a resource. It should never modify data. GET requests are **safe** and **idempotent**.

```go
// GET /api/products - list all products
// GET /api/products/123 - get a single product

mux.HandleFunc("GET /api/products", h.ListProducts)
mux.HandleFunc("GET /api/products/{id}", h.GetProduct)
```

A GET request should not have a body. Parameters go in the URL or query string.

### POST - Create

**POST** creates a new resource. It is neither safe nor idempotent. Sending the same POST twice creates two resources.

```go
// POST /api/products - create a new product

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    product, err := h.service.Create(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated) // 201 Created
    json.NewEncoder(w).Encode(product)
}
```

### PUT - Full Update

**PUT** replaces a resource entirely. The client must send the complete resource. It is **idempotent**: sending the same PUT twice has the same effect as sending it once.

```go
// PUT /api/products/123 - replace product 123 entirely

func (h *ProductHandler) Update(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.ParseInt(r.PathValue("id"), 10, 64)

    var req UpdateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // PUT replaces ALL fields, even if some are empty
    product, err := h.service.Update(id, req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```

### PATCH - Partial Update

**PATCH** updates part of a resource. Only the fields sent in the request are modified. It is not guaranteed to be idempotent.

```go
// PATCH /api/products/123 - update specific fields of product 123

func (h *ProductHandler) PartialUpdate(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.ParseInt(r.PathValue("id"), 10, 64)

    var req map[string]interface{} // Accept any fields
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    product, err := h.service.PartialUpdate(id, req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```

### DELETE - Remove

**DELETE** removes a resource. It is **idempotent**: deleting the same resource twice has the same effect as deleting it once (the resource is gone).

```go
// DELETE /api/products/123 - delete product 123

func (h *ProductHandler) Delete(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.ParseInt(r.PathValue("id"), 10, 64)

    err := h.service.Delete(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.WriteHeader(http.StatusNoContent) // 204 No Content
}
```

### OPTIONS - CORS Preflight

**OPTIONS** asks the server what methods are allowed on a resource. Browsers send this automatically for cross-origin requests.

```go
// Browser sends this before a cross-origin POST
// OPTIONS /api/products
// The server responds with:
// Access-Control-Allow-Methods: GET, POST, PUT, DELETE
```

### HEAD - Like GET but No Body

**HEAD** is identical to GET but the server returns only the headers, not the body. Useful for checking if a resource exists or getting metadata.

```go
// HEAD /api/products/123 - check if product 123 exists
// Returns status 200 if it exists, 404 if not
// No response body
```

## Safe vs Idempotent Methods

| Method | Safe | Idempotent | Description |
|--------|------|------------|-------------|
| **GET** | Yes | Yes | Read only, no side effects |
| **HEAD** | Yes | Yes | Like GET, no body |
| **OPTIONS** | Yes | Yes | Describe communication options |
| **POST** | No | No | Create new resource |
| **PUT** | No | Yes | Replace resource entirely |
| **PATCH** | No | No* | Partial update |
| **DELETE** | No | Yes | Remove resource |

**Safe** means the method does not modify the resource. You can call a safe method without worrying about side effects.

**Idempotent** means calling the method multiple times has the same effect as calling it once. PUT is idempotent because replacing a resource with the same data gives the same result. POST is not idempotent because each call creates a new resource.

*PATCH can be idempotent depending on implementation, but it is not guaranteed.

## RESTful Conventions

Using HTTP methods correctly follows **REST** conventions:

```
GET    /api/products       -> List all products
POST   /api/products       -> Create a new product
GET    /api/products/123   -> Get product 123
PUT    /api/products/123   -> Replace product 123
PATCH  /api/products/123   -> Update part of product 123
DELETE /api/products/123   -> Delete product 123
```

The URL is always a **noun** (products), never a **verb** (getProduct, deleteProduct). The HTTP method is the verb.

Bad:
```
GET /api/getProducts
POST /api/createProduct
POST /api/deleteProduct
```

Good:
```
GET /api/products
POST /api/products
DELETE /api/products/123
```

## Proper Status Codes

Each method should return the appropriate HTTP status code:

| Method | Success Code | Meaning |
|--------|-------------|---------|
| **GET** | 200 OK | Resource found and returned |
| **POST** | 201 Created | New resource created |
| **PUT** | 200 OK | Resource updated |
| **PATCH** | 200 OK | Resource partially updated |
| **DELETE** | 204 No Content | Resource deleted, no body |

## Route Registration in Go

Using Go 1.22+ routing with method-based patterns:

```go
func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/products", h.ListProducts)
    mux.HandleFunc("POST /api/products", h.CreateProduct)
    mux.HandleFunc("GET /api/products/{id}", h.GetProduct)
    mux.HandleFunc("PUT /api/products/{id}", h.UpdateProduct)
    mux.HandleFunc("PATCH /api/products/{id}", h.PartialUpdate)
    mux.HandleFunc("DELETE /api/products/{id}", h.DeleteProduct)

    log.Println("Server running on :8080")
    http.ListenAndServe(":8080", mux)
}
```

Go 1.22 added method-based routing to the standard library. Before that, you needed a third-party router like chi or gorilla/mux.

## My Note on HTTP Methods

I used to use POST for everything. Create? POST. Update? POST. Delete? POST. It worked, but it was confusing. Other developers had to read my code to understand what each endpoint did.

When I started using proper HTTP methods, everything clicked. The API became self-documenting. A DELETE request clearly means "delete this resource." A GET request clearly means "read this resource." No ambiguity.

Use the right method for the right action. It makes your API predictable, and predictable APIs are easy to use.
