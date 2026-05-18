# Middleware in Go

Middleware is one of those concepts that sounds fancy but is actually simple once you see it in action. And it is incredibly powerful. Once I understood middleware, the way I build web servers changed completely.

## What is Middleware?

A **middleware** is a function that wraps around your handler. It runs code before the handler, then passes the request to the handler, and can also run code after the handler.

Think of it like a security checkpoint at an airport. Every passenger (request) goes through the checkpoint (middleware) before boarding the plane (handler). The checkpoint can:
- Let the passenger through
- Stop the passenger and send them away
- Add something to the passenger (like a stamp on their passport)

The most common pattern in Go is:

```go
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Code BEFORE the handler runs
        
        next.ServeHTTP(w, r)  // Call the next handler
        
        // Code AFTER the handler runs
    })
}
```

The middleware takes the next handler as an argument, wraps it in a new handler, and returns that wrapper. The `next.ServeHTTP(w, r)` call is where the actual handler runs.

## How Middleware Works: Russian Nesting Dolls

I like to think of middleware as **Russian nesting dolls**. Each middleware wraps around the next one. When a request comes in, it goes through the outermost doll first, then the next one, then the next, until it reaches the actual handler at the center.

```
Request -> [Logging] -> [Recovery] -> [Auth] -> [Handler] -> Response
```

When the handler finishes, the response goes back through the dolls in reverse order. This means each middleware can do something both before AND after the handler.

## Logging Middleware

Let us start with the most useful middleware: **logging**. Every request should be logged so you can debug problems.

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Log the incoming request
        fmt.Printf("[%s] %s %s - Started\n",
            r.Method,
            r.URL.Path,
            r.RemoteAddr,
        )
        
        // Call the next handler
        next.ServeHTTP(w, r)
        
        // Log after the handler finishes
        duration := time.Since(start)
        fmt.Printf("[%s] %s - Completed in %v\n",
            r.Method,
            r.URL.Path,
            duration,
        )
    })
}
```

Now every request gets logged with how long it took. Super useful for finding slow endpoints.

## CORS Middleware

We already touched on CORS, but let us make it a proper middleware:

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "86400")
        
        // Handle preflight requests
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return  // Do not call next handler for preflight
        }
        
        next.ServeHTTP(w, r)
    })
}
```

Notice how the CORS middleware can **stop the chain** by returning early. For preflight requests, we set headers and return without calling `next.ServeHTTP()`. The actual handler never runs. This is a key feature of middleware: it can decide whether to pass the request along or handle it completely.

## Recovery Middleware

What happens if your handler panics? Without recovery, the server crashes. With recovery middleware, the panic is caught and a 500 error is returned instead.

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Defer a function to catch panics
        defer func() {
            if err := recover(); err != nil {
                fmt.Printf("PANIC: %v\n", err)
                fmt.Println(string(debug.Stack()))
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

The `defer` and `recover()` pattern catches any panic that happens in the handler or any middleware after this one. The server stays running and the client gets a proper 500 error instead of a connection reset.

## Chaining Multiple Middlewares

Middleware becomes really powerful when you chain them together. With chi, it looks like this:

```go
func main() {
    r := chi.NewRouter()
    
    // Apply middleware in order
    r.Use(recoveryMiddleware)   // 1st: catch panics
    r.Use(loggingMiddleware)    // 2nd: log requests
    r.Use(corsMiddleware)       // 3rd: handle CORS
    
    r.Get("/products", listProducts)
    r.Post("/products", createProduct)
    
    http.ListenAndServe(":8080", r)
}
```

The order matters. A request goes through recovery first, then logging, then CORS, then the handler. If logging middleware panics, the recovery middleware catches it. If CORS rejects a preflight, logging has already logged the request.

Think about the order carefully. Generally:
1. **Recovery** should be first (outermost) so it catches panics from everything
2. **Logging** should be early so all requests are logged
3. **CORS** should be before your handlers
4. **Auth** should be after CORS but before handlers

## A Custom Response Writer for Better Logging

Our logging middleware is nice, but it does not log the status code. That is because `http.ResponseWriter` does not expose the status code after it is set. We need a custom wrapper:

```go
type responseRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrap the response writer
        recorder := &responseRecorder{
            ResponseWriter: w,
            statusCode:     http.StatusOK,  // default
        }
        
        next.ServeHTTP(recorder, r)
        
        duration := time.Since(start)
        fmt.Printf("[%s] %s %s - %d (%v)\n",
            r.Method,
            r.URL.Path,
            r.RemoteAddr,
            recorder.statusCode,
            duration,
        )
    })
}
```

The `responseRecorder` wraps the original `ResponseWriter` and records the status code when `WriteHeader` is called. This is a common pattern in Go middleware. You wrap the response writer to intercept and record information.

## Complete Middleware Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "runtime/debug"
    "time"

    "github.com/go-chi/chi/v5"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

var products = []Product{{ID: 1, Name: "Go Book", Price: 39.99}}

func main() {
    r := chi.NewRouter()
    
    r.Use(recoveryMiddleware)
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware)
    
    r.Get("/products", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(products)
    })
    
    r.Get("/panic", func(w http.ResponseWriter, r *http.Request) {
        panic("something went wrong!")
    })
    
    fmt.Println("Server on :8080")
    http.ListenAndServe(":8080", r)
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                fmt.Printf("PANIC recovered: %v\n%s\n", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        recorder := &responseRecorder{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(recorder, r)
        fmt.Printf("[%s] %s -> %d (%v)\n", r.Method, r.URL.Path, recorder.statusCode, time.Since(start))
    })
}

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        next.ServeHTTP(w, r)
    })
}

type responseRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}
```

Try visiting `/panic`. Instead of crashing the server, you get a 500 error and the panic is logged. That is the power of middleware.

Middleware is not optional for production servers. Logging, recovery, and CORS should be in every project. Once you understand the pattern, you will write custom middleware for all sorts of things: authentication, rate limiting, request tracing, and more.
