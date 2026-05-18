# POST Request in Go

If GET is for reading data, **POST** is for creating data. When you submit a form, when you click "Add to Cart", when you create a new account, that is usually a POST request.

POST requests send data to the server in the **request body**. The body can be JSON, form data, XML, or any format. For APIs, JSON is the standard.

Let us see how to handle POST requests in Go.

## Reading the Request Body

The most important thing about POST requests is reading the body. Here is how you do it with JSON:

```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    var product models.Product
    err := json.NewDecoder(r.Body).Decode(&product)
    if err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Use the product...
    product.ID = nextID
    nextID++
    products = append(products, product)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}
```

Let me break down the key line:

```go
json.NewDecoder(r.Body).Decode(&product)
```

- `r.Body` is the request body, which implements `io.Reader`
- `json.NewDecoder()` creates a JSON decoder that reads from that body
- `.Decode(&product)` reads the JSON and puts it into our `product` struct
- We pass `&product` (a pointer) so that `Decode` can modify the struct

This one line replaces what would be dozens of lines of manual parsing in some other languages.

## Validating Input

Never trust user input. Ever. Even if your frontend validates the data, someone can always send a request directly to your API using curl or Postman.

Here is a simple validation function:

```go
func validateProduct(p models.Product) map[string]string {
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

And here is how you use it in your handler:

```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    var product models.Product
    err := json.NewDecoder(r.Body).Decode(&product)
    if err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate
    validationErrors := validateProduct(product)
    if len(validationErrors) > 0 {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "errors": validationErrors,
        })
        return
    }
    
    // All good, create the product
    product.ID = nextID
    nextID++
    products = append(products, product)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}
```

Now if someone sends an empty product, they get back a helpful error message:

```json
{
    "errors": {
        "name": "Name is required",
        "price": "Price must be greater than zero",
        "category": "Category is required"
    }
}
```

## Returning 201 Created

When you create a new resource, you should return **201 Created** instead of 200 OK. This is the standard HTTP convention. It tells the client, "I created something new for you."

```go
w.WriteHeader(http.StatusCreated)  // 201
```

It is also good practice to return the created resource in the response body, including the server-assigned ID. This way, the client knows what was created without making another request.

## Common Mistake: Not Closing the Request Body

You will see a lot of tutorials that do not close `r.Body`. Technically, for `json.NewDecoder().Decode()`, Go handles this for you. But if you are using `io.ReadAll` to read the raw body, you should close it:

```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    // Read raw body bytes
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Failed to read body", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    var product models.Product
    err = json.Unmarshal(body, &product)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    // Process the product...
}
```

When should you use `io.ReadAll` vs `json.NewDecoder`?

- **`json.NewDecoder(r.Body).Decode(&product)`** - Use this when you just want to parse JSON into a struct. This is the common case.
- **`io.ReadAll(r.Body)` + `json.Unmarshal()`** - Use this when you need the raw body bytes for something else, like logging or signature verification.

## Handling Different Content Types

Sometimes the client might send data in a different format. You can check the `Content-Type` header:

```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    contentType := r.Header.Get("Content-Type")
    if contentType != "application/json" {
        http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
        return
    }
    
    var product models.Product
    err := json.NewDecoder(r.Body).Decode(&product)
    if err != nil {
        http.Error(w, "Invalid JSON body", http.StatusBadRequest)
        return
    }
    
    // Continue processing...
}
```

## Complete POST Example

Here is a full, working example of handling a POST request:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
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
        json.NewEncoder(w).Encode(products)
        
    case http.MethodPost:
        var product Product
        if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
            http.Error(w, `{"error": "Invalid JSON"}`, http.StatusBadRequest)
            return
        }
        
        // Validate
        if product.Name == "" {
            http.Error(w, `{"error": "Name is required"}`, http.StatusBadRequest)
            return
        }
        if product.Price <= 0 {
            http.Error(w, `{"error": "Price must be positive"}`, http.StatusBadRequest)
            return
        }
        
        // Create
        product.ID = nextID
        nextID++
        products = append(products, product)
        
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(product)
        fmt.Printf("Created product: %+v\n", product)
        
    default:
        http.Error(w, `{"error": "Method not allowed"}`, http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/products", productsHandler)
    fmt.Println("Server on :8080")
    http.ListenAndServe(":8080", nil)
}
```

Test it:

```bash
# Create a product
curl -X POST http://localhost:8080/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Wireless Mouse", "description": "No cables", "price": 29.99, "category": "Electronics"}'

# Try with missing name (should fail)
curl -X POST http://localhost:8080/products \
  -H "Content-Type: application/json" \
  -d '{"description": "No name", "price": 10.00, "category": "Test"}'
```

POST requests are how you accept data from the client. Always validate, always return proper status codes, and always return the created resource. These habits will save you from countless bugs down the road.
