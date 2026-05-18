# Server Development (Summary)

## Putting It All Together

This is the culmination of everything we have learned. Routing, middleware, handlers, services, repositories, databases, authentication, error handling, logging, and graceful shutdown. All combined into one production-ready Go server.

When I wrote my first Go server, it was a single file with everything in main(). Handlers talked directly to the database. There was no error handling. No logging. No graceful shutdown. It worked, but only for a weekend project.

A production server is different. It needs to be reliable, maintainable, and observable. Let me show you how to build one.

## Project Structure

```
myapp/
├── cmd/
│   └── server/
│       └── main.go
├── domain/
│   ├── product/
│   │   ├── product.go
│   │   └── repository.go
│   └── user/
│       ├── user.go
│       └── repository.go
├── usecase/
│   ├── product_service.go
│   └── user_service.go
├── adapter/
│   ├── handler/
│   │   ├── product_handler.go
│   │   └── user_handler.go
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── logging.go
│   │   ├── cors.go
│   │   └── recovery.go
│   └── repository/
│       └── postgres/
│           ├── product_repo.go
│           └── user_repo.go
├── pkg/
│   ├── database/
│   │   └── connect.go
│   ├── response/
│   │   └── response.go
│   └── config/
│       └── config.go
├── migrations/
│   ├── 000001_create_users.up.sql
│   └── 000001_create_users.down.sql
├── go.mod
└── go.sum
```

## Database Connection

```go
// pkg/database/connect.go
package database

import (
    "database/sql"
    "fmt"
    "os"
    "time"

    _ "github.com/lib/pq"
)

func Connect() (*sql.DB, error) {
    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=%s",
        env("DB_USER", "postgres"),
        env("DB_PASSWORD", ""),
        env("DB_HOST", "localhost"),
        env("DB_PORT", "5432"),
        env("DB_NAME", "myapp"),
        env("DB_SSLMODE", "disable"),
    )

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return db, nil
}

func env(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

## Middleware

```go
// adapter/middleware/logging.go
package middleware

import (
    "log"
    "net/http"
    "time"
)

func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// adapter/middleware/recovery.go
package middleware

import (
    "log"
    "net/http"
    "runtime/debug"
)

func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// adapter/middleware/cors.go
package middleware

import "net/http"

func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// adapter/middleware/auth.go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
)

func Auth(secret []byte, next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        header := r.Header.Get("Authorization")
        if header == "" {
            http.Error(w, `{"error":"missing authorization"}`, http.StatusUnauthorized)
            return
        }

        parts := strings.SplitN(header, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, `{"error":"invalid authorization format"}`, http.StatusUnauthorized)
            return
        }

        token, err := jwt.Parse(parts[1], func(t *jwt.Token) (interface{}, error) {
            return secret, nil
        })
        if err != nil || !token.Valid {
            http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
            return
        }

        claims := token.Claims.(jwt.MapClaims)
        ctx := context.WithValue(r.Context(), "user_id", int64(claims["user_id"].(float64)))
        next(w, r.WithContext(ctx))
    }
}
```

## Response Helper

```go
// pkg/response/response.go
package response

import (
    "encoding/json"
    "net/http"
)

type APIResponse struct {
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Code    string      `json:"code,omitempty"`
    Message string      `json:"message,omitempty"`
}

func JSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(APIResponse{Data: data})
}

func Error(w http.ResponseWriter, status int, code, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(APIResponse{
        Error:   message,
        Code:    code,
        Message: message,
    })
}
```

## The Main File

```go
// cmd/server/main.go
package main

import (
    "context"
    "database/sql"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "myapp/adapter/handler"
    "myapp/adapter/middleware"
    pg "myapp/adapter/repository/postgres"
    "myapp/domain/product"
    "myapp/domain/user"
    "myapp/pkg/database"
    "myapp/usecase"
)

func main() {
    // Connect to database
    db, err := database.Connect()
    if err != nil {
        log.Fatalf("Database connection failed: %v", err)
    }
    defer db.Close()
    log.Println("Connected to database")

    // Create repositories
    var productRepo product.Repository = pg.NewProductRepo(db)
    var userRepo user.Repository = pg.NewUserRepo(db)

    // Create services
    productService := usecase.NewProductService(productRepo)
    userService := usecase.NewUserService(userRepo)

    // Create handlers
    productHandler := handler.NewProductHandler(productService)
    userHandler := handler.NewUserHandler(userService)

    // Define routes
    mux := http.NewServeMux()

    // Public routes
    mux.HandleFunc("POST /api/auth/login", userHandler.Login)
    mux.HandleFunc("POST /api/auth/register", userHandler.Register)

    // Protected routes
    jwtSecret := []byte(os.Getenv("JWT_SECRET"))
    mux.HandleFunc("GET /api/products", middleware.Auth(jwtSecret, productHandler.List))
    mux.HandleFunc("POST /api/products", middleware.Auth(jwtSecret, productHandler.Create))
    mux.HandleFunc("GET /api/products/{id}", middleware.Auth(jwtSecret, productHandler.Get))
    mux.HandleFunc("PUT /api/products/{id}", middleware.Auth(jwtSecret, productHandler.Update))
    mux.HandleFunc("DELETE /api/products/{id}", middleware.Auth(jwtSecret, productHandler.Delete))

    // Apply global middleware
    var h http.Handler = mux
    h = middleware.CORS(h)
    h = middleware.Logging(h)
    h = middleware.Recovery(h)

    // Create HTTP server
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      h,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Printf("Received %v, shutting down...\n", sig)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("Forced shutdown: %v", err)
    }

    log.Println("Server stopped gracefully")
}
```

## What Makes This Production-Ready

**Layered architecture** - Domain, use cases, adapters, and infrastructure are separated. Each layer has a clear responsibility.

**Dependency injection** - All dependencies are created in main() and injected. Nothing creates its own dependencies.

**Middleware stack** - CORS, logging, panic recovery, and authentication are applied consistently to all routes.

**Error handling** - Every layer returns errors. Handlers convert errors to appropriate HTTP responses.

**Graceful shutdown** - The server finishes processing current requests before exiting. No connections are dropped.

**Configuration** - All settings come from environment variables. No hardcoded values.

**Connection pooling** - The database connection pool is properly configured with limits and timeouts.

**Timeouts** - The HTTP server has read, write, and idle timeouts. No hanging connections.

## My Reflection on the Journey

I started this book with Hello World. A simple program that prints one line. Now we have built a complete, production-ready Go server with authentication, database operations, clean architecture, and graceful shutdown.

The journey from Hello World to this server was not a straight line. There were many detours. I had to learn about HTTP before I could build handlers. I had to learn about interfaces before I could decouple my code. I had to learn about concurrency before I could handle thousands of requests.

But each piece built on the last. HTTP handlers led to middleware. Middleware led to authentication. Authentication led to JWT. JWT led to secure APIs. Secure APIs led to production deployment.

That is the nature of learning. You do not learn everything at once. You learn one thing, then another, then how they connect. Eventually, the pieces form a complete picture.

This server is not the end. It is the beginning. From here, you can add caching, rate limiting, background jobs, websockets, observability, and much more. But the foundation is solid. Everything you add will fit into this structure.

Go is a beautiful language for backend development. It is simple, fast, and pragmatic. The concurrency model is elegant. The standard library is powerful. The community is welcoming. I am glad I learned it, and I hope you are too.

Keep building. Keep learning. Keep shipping.
