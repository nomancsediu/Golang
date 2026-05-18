# OPTIONS Method

In the last chapter, we talked about preflight requests and how the browser sends an OPTIONS request before certain cross-origin requests. Now let us look at the **OPTIONS method** more closely, because there is more to it than just CORS.

## What is the OPTIONS Method?

The **OPTIONS** HTTP method is a way for a client to ask the server, "What can I do with this resource?" It is like walking up to a restaurant host and asking, "What do you serve here?" before you actually order.

Here is what an OPTIONS request looks like:

```
OPTIONS /products HTTP/1.1
Host: api.example.com
```

And the response tells the client what methods are allowed:

```
HTTP/1.1 204 No Content
Allow: GET, POST, OPTIONS
```

The `Allow` header lists all the HTTP methods that this endpoint supports. This is the original purpose of OPTIONS, before CORS made it famous.

## OPTIONS in CORS Preflight

When a browser sends a preflight request, it sends an OPTIONS request with extra headers:

```
OPTIONS /products HTTP/1.1
Host: api.example.com
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

The browser is asking two questions:
1. Is this origin allowed to make requests?
2. Is this method and these headers allowed?

The server answers with CORS headers:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

So in the context of CORS, OPTIONS is the mechanism the browser uses to check permissions before sending the actual request.

## Handling OPTIONS in Go

Let us look at different ways to handle OPTIONS requests in Go.

### Method 1: Handle in Each Handler

The simplest approach is to check for OPTIONS in each handler:

```go
func productsHandler(w http.ResponseWriter, r *http.Request) {
    // Handle OPTIONS first
    if r.Method == http.MethodOptions {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
        w.WriteHeader(http.StatusNoContent)
        return
    }
    
    // Handle other methods
    w.Header().Set("Content-Type", "application/json")
    
    switch r.Method {
    case http.MethodGet:
        json.NewEncoder(w).Encode(products)
    case http.MethodPost:
        var p Product
        json.NewDecoder(r.Body).Decode(&p)
        p.ID = len(products) + 1
        products = append(products, p)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(p)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}
```

This works but gets repetitive. Every handler needs the same OPTIONS check.

### Method 2: Central CORS Middleware

A much better approach is to handle OPTIONS in a central middleware:

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers for all responses
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "86400")
        
        // If this is a preflight, respond immediately
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        
        // Otherwise, pass to the next handler
        next.ServeHTTP(w, r)
    })
}
```

Now every request goes through this middleware first. If it is OPTIONS, we handle it and return. Otherwise, we pass it to the actual handler. Clean and DRY.

### Method 3: Dedicated OPTIONS Handler

Some people prefer to register OPTIONS handlers explicitly:

```go
func optionsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
    w.Header().Set("Access-Control-Max-Age", "86400")
    w.WriteHeader(http.StatusNoContent)
}

func main() {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/products", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodOptions:
            optionsHandler(w, r)
        case http.MethodGet:
            getProducts(w, r)
        case http.MethodPost:
            createProduct(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    })
    
    http.ListenAndServe(":8080", mux)
}
```

This is more explicit but also more verbose. I prefer the middleware approach.

## Understanding the CORS Headers

Let me go deeper into each CORS-related header:

**`Access-Control-Allow-Origin`** - Specifies which origin can access the resource. Use `*` to allow any origin, or specify a single origin like `http://localhost:3000`. You cannot list multiple origins; you either use `*` or one specific origin.

**`Access-Control-Allow-Methods`** - Lists the HTTP methods the client is allowed to use. Include all methods your API supports, including OPTIONS.

**`Access-Control-Allow-Headers`** - Lists the custom headers the client is allowed to send. Common ones include `Content-Type`, `Authorization`, and `X-Requested-With`.

**`Access-Control-Expose-Headers`** - By default, JavaScript can only read a few basic response headers. If you want the client to read custom headers, list them here.

**`Access-Control-Max-Age`** - How long (in seconds) the browser should cache the preflight result. Setting this to `86400` (24 hours) reduces the number of preflight requests.

**`Access-Control-Allow-Credentials`** - Set to `true` if the client needs to send cookies or authentication headers. When this is `true`, you cannot use `*` for the origin.

## A Complete Working Example

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

var products = []Product{{ID: 1, Name: "Go Book", Price: 39.99}}

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")
        
        // In production, check against allowed origins
        allowedOrigins := []string{
            "http://localhost:3000",
            "http://localhost:5173",
        }
        
        for _, o := range allowedOrigins {
            if origin == o {
                w.Header().Set("Access-Control-Allow-Origin", o)
                break
            }
        }
        
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Allow-Credentials", "true")
        w.Header().Set("Access-Control-Max-Age", "86400")
        
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

func productsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    switch r.Method {
    case http.MethodGet:
        json.NewEncoder(w).Encode(products)
    case http.MethodPost:
        var p Product
        if err := json.NewDecoder(r.Body).Decode(&p); err != nil {
            http.Error(w, "Invalid body", http.StatusBadRequest)
            return
        }
        p.ID = len(products) + 1
        products = append(products, p)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(p)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/products", productsHandler)
    
    fmt.Println("Server on :8080 with CORS support")
    http.ListenAndServe(":8080", corsMiddleware(mux))
}
```

The OPTIONS method might seem like a niche topic, but if you are building APIs that browsers will call, you will deal with it constantly. Understanding it saves hours of debugging "why does my API work in curl but not in the browser."
