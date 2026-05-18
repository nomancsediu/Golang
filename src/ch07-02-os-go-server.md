# OS or Go Server: Building Your First Server

One of the things I love about Go is that you do not need any external framework to build a web server. The standard library has everything you need. In Python, you might reach for Flask or Django. In Node.js, you use Express. But in Go, the built-in `net/http` package is genuinely production-ready.

Let me show you how to build a basic server from scratch.

## The Simplest Server

Here is the smallest possible Go web server:

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    fmt.Println("Server starting on port 8080...")
    http.ListenAndServe(":8080", nil)
}
```

Run this, open your browser to `http://localhost:8080`, and you will see "Hello, World!". That is it. You just built a web server in about 10 lines of code.

Let me break down what is happening here.

## Understanding the Pieces

**`http.HandleFunc`** - This registers a function to handle requests at a specific path. The first argument is the path (`"/"`), and the second is the handler function.

**`http.ResponseWriter`** - This is how you write the response back to the client. Think of it as a pipe that sends data from your server to the user's browser.

**`*http.Request`** - This contains everything about the incoming request: the URL, headers, query parameters, body, and HTTP method.

**`http.ListenAndServe(":8080", nil)`** - This starts the server on port 8080. The second argument (`nil`) tells Go to use the **DefaultServeMux**, which is the default request router.

## What is the DefaultServeMux?

The **DefaultServeMux** is Go's built-in request multiplexer. It maps URL paths to handler functions. When you call `http.HandleFunc`, you are adding a route to this default multiplexer.

```go
// These all register routes on the DefaultServeMux
http.HandleFunc("/", homeHandler)
http.HandleFunc("/about", aboutHandler)
http.HandleFunc("/contact", contactHandler)
```

When a request comes in, the DefaultServeMux checks the path and calls the matching handler. If no exact match is found, it falls back to the `/` handler.

## Handling Multiple Routes

Let us build a slightly more interesting server with multiple routes:

```go
package main

import (
    "fmt"
    "net/http"
)

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to the home page!")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This is the about page.")
}

func apiHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintf(w, `{"message": "Hello from the API"}`)
}

func main() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)
    http.HandleFunc("/api", apiHandler)

    fmt.Println("Server starting on port 8080...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Printf("Server failed: %v\n", err)
    }
}
```

Notice how each handler does something different. The `apiHandler` sets the `Content-Type` header to `application/json` and returns JSON data. This is the beginning of building a real API.

## How Go Handles Requests Internally

This is the part that blew my mind when I first learned it.

When Go's HTTP server receives a request, it spawns a **new goroutine** to handle that request. This means every single request runs in its own lightweight thread.

Why does this matter? Let me compare:

**Node.js approach:** Node uses a single-threaded event loop. It can handle many connections, but if one request does something slow (like a database query), it can block other requests unless you use async patterns carefully.

**Go approach:** Each request gets its own goroutine. Goroutines are extremely cheap (they start with about 2KB of stack memory). You can easily have thousands or even millions of goroutines running at the same time. If one request takes a long time, it does not block the others.

This is why Go servers scale so well. You get concurrency for free, without the complexity of async/await patterns.

## A More Complete Example

Let us write a server that returns different content types and handles basic routing:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type Response struct {
    Status  string `json:"status"`
    Message string `json:"message"`
    Time    string `json:"time"`
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    resp := Response{
        Status:  "ok",
        Message: "Server is running",
        Time:    time.Now().Format(time.RFC3339),
    }
    
    json.NewEncoder(w).Encode(resp)
}

func echoHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "stranger"
    }
    
    resp := Response{
        Status:  "ok",
        Message: fmt.Sprintf("Hello, %s!", name),
        Time:    time.Now().Format(time.RFC3339),
    }
    
    json.NewEncoder(w).Encode(resp)
}

func main() {
    http.HandleFunc("/health", healthHandler)
    http.HandleFunc("/echo", echoHandler)

    fmt.Println("Server starting on port 8080...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Printf("Server error: %v\n", err)
    }
}
```

Now try these URLs:
- `http://localhost:8080/health` returns the server status
- `http://localhost:8080/echo?name=Go` returns a greeting

## Limitations of the DefaultServeMux

The default mux works fine for small projects, but it has limitations:

- **No path parameters** - You cannot do `/users/{id}` easily
- **No method matching** - You cannot differentiate between GET `/users` and POST `/users`
- **No route groups** - You cannot group routes under a common prefix with shared middleware

We will solve these problems later with third-party routers like **chi**. For now, the standard library is more than enough to get started.

The key takeaway is this: Go gives you a fully functional web server out of the box. No frameworks needed. No magic. Just clean, simple code that does exactly what you tell it to do.
