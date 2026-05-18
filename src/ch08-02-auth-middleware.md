# Authentication Middleware

## Why We Need Auth Middleware

Now that we can create and verify JWT tokens, we need a way to **check every request** for a valid token. We do not want to add token verification code to every single handler. That would be repetitive and error-prone.

The solution is **middleware**. Auth middleware sits between the incoming request and your handler. It checks the token. If the token is valid, it passes the request to the handler. If the token is invalid, it returns **401 Unauthorized** immediately.

This way, you write the auth check once and apply it to as many routes as you want.

## Extracting JWT from the Authorization Header

The client sends the JWT in the **Authorization** header like this:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

The format is always `Bearer <token>`. So we need to:

1. Get the Authorization header
2. Check that it starts with "Bearer "
3. Extract the token part after "Bearer "

Here is how to do that:

```go
func ExtractToken(r *http.Request) (string, error) {
    // Get the Authorization header
    authHeader := r.Header.Get("Authorization")
    if authHeader == "" {
        return "", fmt.Errorf("authorization header is missing")
    }

    // Check the format: "Bearer <token>"
    parts := strings.SplitN(authHeader, " ", 2)
    if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
        return "", fmt.Errorf("authorization header format must be Bearer <token>")
    }

    return parts[1], nil
}
```

## Building the Auth Middleware

Here is the complete auth middleware:

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "strings"
)

// Context key for user info
type contextKey string

const (
    ContextKeyUserID contextKey = "user_id"
    ContextKeyEmail  contextKey = "email"
)

// AuthMiddleware checks for a valid JWT on every request
func AuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Extract token from Authorization header
        tokenString, err := ExtractToken(r)
        if err != nil {
            http.Error(w, `{"error": "unauthorized: `+err.Error()+`"}`,
                http.StatusUnauthorized)
            return
        }

        // Verify the token
        claims, err := VerifyToken(tokenString)
        if err != nil {
            http.Error(w, `{"error": "unauthorized: invalid token"}`,
                http.StatusUnauthorized)
            return
        }

        // Add user info to the request context
        ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.UserID)
        ctx = context.WithValue(ctx, ContextKeyEmail, claims.Email)

        // Call the next handler with the updated context
        next(w, r.WithContext(ctx))
    }
}
```

The middleware extracts the token, verifies it, and then adds the user info to the request **context**. Handlers downstream can access the user info from the context.

## Using User Info in Handlers

In your protected handlers, you can get the user info from the context:

```go
// GetUserID extracts user ID from the request context
func GetUserID(ctx context.Context) (int64, error) {
    userID, ok := ctx.Value(ContextKeyUserID).(int64)
    if !ok {
        return 0, fmt.Errorf("user ID not found in context")
    }
    return userID, nil
}

// Example protected handler
func GetProfileHandler(w http.ResponseWriter, r *http.Request) {
    // Get user ID from context (set by auth middleware)
    userID, err := GetUserID(r.Context())
    if err != nil {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    // Fetch user from database
    user, err := FindUserByID(userID)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

## Public vs Protected Routes

Not every route needs auth. Login and register should be public. Everything else should be protected:

```go
func main() {
    mux := http.NewServeMux()

    // Public routes (no auth needed)
    mux.HandleFunc("/api/login", LoginHandler)
    mux.HandleFunc("/api/register", RegisterHandler)

    // Protected routes (auth required)
    mux.HandleFunc("/api/profile", AuthMiddleware(GetProfileHandler))
    mux.HandleFunc("/api/products", AuthMiddleware(ListProductsHandler))
    mux.HandleFunc("/api/orders", AuthMiddleware(ListOrdersHandler))

    fmt.Println("Server running on :8080")
    http.ListenAndServe(":8080", mux)
}
```

## Complete Example: Ecommerce API with Auth

Here is a more complete example with multiple protected routes:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("my-super-secret-key")

type Claims struct {
    UserID int64  `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

// Auth middleware
func RequireAuth(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            respondJSON(w, http.StatusUnauthorized,
                map[string]string{"error": "missing authorization header"})
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            respondJSON(w, http.StatusUnauthorized,
                map[string]string{"error": "invalid authorization format"})
            return
        }

        token, err := jwt.ParseWithClaims(parts[1], &Claims{},
            func(t *jwt.Token) (interface{}, error) {
                return jwtSecret, nil
            })

        if err != nil || !token.Valid {
            respondJSON(w, http.StatusUnauthorized,
                map[string]string{"error": "invalid or expired token"})
            return
        }

        claims := token.Claims.(*Claims)
        ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.UserID)
        next(w, r.WithContext(ctx))
    }
}

// Helper to send JSON responses
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func main() {
    mux := http.NewServeMux()

    // Public
    mux.HandleFunc("/api/auth/login", LoginHandler)
    mux.HandleFunc("/api/auth/register", RegisterHandler)

    // Protected
    mux.HandleFunc("/api/profile", RequireAuth(GetProfileHandler))
    mux.HandleFunc("/api/orders", RequireAuth(ListOrdersHandler))
    mux.HandleFunc("/api/orders/create", RequireAuth(CreateOrderHandler))

    fmt.Println("Ecommerce API running on :8080")
    http.ListenAndServe(":8080", mux)
}
```

## My Experience with Auth Middleware

When I first learned about middleware, it felt like magic. You wrap your handler with another function, and suddenly every request goes through the auth check. No need to repeat code.

The biggest mistake I made early on was forgetting to pass the updated context to the next handler. I would add the user ID to the context but then call `next(w, r)` instead of `next(w, r.WithContext(ctx))`. The handler downstream would not find the user info. Always remember to pass the updated context.

Another thing I learned: **always return proper error messages**. Do not just say "unauthorized". Tell the client what went wrong. Was the token missing? Was it expired? Was the format wrong? This makes debugging so much easier.

Auth middleware is one of the most powerful patterns in web development. Write it once, apply it everywhere.
