# Advanced Middleware

Now that you understand the basics of middleware, let us go deeper. Real production servers need more than just logging and CORS. They need **request tracking**, **rate limiting**, **authentication**, and **performance monitoring**. All of these can be implemented as middleware.

## Context-Based Middleware

Before we write more middleware, you need to understand **request context**. Every `http.Request` in Go has a `Context()` attached to it. Context is how you pass data through the middleware chain without using global variables.

Here is the pattern:

```go
func someMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add a value to the request context
        ctx := context.WithValue(r.Context(), "myKey", "myValue")
        
        // Create a new request with the updated context
        r = r.WithContext(ctx)
        
        next.ServeHTTP(w, r)
    })
}
```

Then in your handler or another middleware, you can read that value:

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    value := r.Context().Value("myKey")
    fmt.Println(value)  // "myValue"
}
```

**Important:** Using string keys like `"myKey"` is error-prone. It is better to define your own **custom type** for context keys:

```go
// Define a custom type for context keys
type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userIDKey    contextKey = "userID"
)
```

This prevents key collisions with other packages that might also store values in the context.

## Request ID Middleware

When debugging issues in production, you need a way to trace a specific request through all your logs. A **request ID** is a unique identifier attached to every request.

```go
type contextKey string
const requestIDKey contextKey = "requestID"

func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Check if a request ID was already set (e.g., by a load balancer)
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            // Generate a new one
            id = generateRequestID()
        }
        
        // Store in context
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        r = r.WithContext(ctx)
        
        // Also set it as a response header
        w.Header().Set("X-Request-ID", id)
        
        next.ServeHTTP(w, r)
    })
}

func generateRequestID() string {
    // Simple ID generation using timestamp and random number
    return fmt.Sprintf("%d-%06d", time.Now().UnixNano(), rand.Intn(1000000))
}

// Helper function to get request ID from context
func GetRequestID(r *http.Request) string {
    if id, ok := r.Context().Value(requestIDKey).(string); ok {
        return id
    }
    return "unknown"
}
```

Now every request has a unique ID. You can include it in your logs:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        reqID := GetRequestID(r)
        
        fmt.Printf("[%s] %s %s - Started\n", reqID, r.Method, r.URL.Path)
        
        next.ServeHTTP(w, r)
        
        fmt.Printf("[%s] %s %s - Completed in %v\n",
            reqID, r.Method, r.URL.Path, time.Since(start))
    })
}
```

## Rate Limiting Middleware

Rate limiting prevents abuse by restricting how many requests a client can make in a given time period. Here is a simple implementation:

```go
type rateLimiter struct {
    visitors map[string]*visitorInfo
    mu       sync.Mutex
    rate     int           // Max requests
    window   time.Duration // Time window
}

type visitorInfo struct {
    count    int
    lastSeen time.Time
}

func newRateLimiter(rate int, window time.Duration) *rateLimiter {
    rl := &rateLimiter{
        visitors: make(map[string]*visitorInfo),
        rate:     rate,
        window:   window,
    }
    
    // Clean up old entries periodically
    go rl.cleanup()
    return rl
}

func (rl *rateLimiter) allow(ip string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    v, exists := rl.visitors[ip]
    if !exists || time.Since(v.lastSeen) > rl.window {
        rl.visitors[ip] = &visitorInfo{count: 1, lastSeen: time.Now()}
        return true
    }
    
    v.lastSeen = time.Now()
    v.count++
    
    return v.count <= rl.rate
}

func (rl *rateLimiter) cleanup() {
    for {
        time.Sleep(time.Minute)
        rl.mu.Lock()
        for ip, v := range rl.visitors {
            if time.Since(v.lastSeen) > rl.window {
                delete(rl.visitors, ip)
            }
        }
        rl.mu.Unlock()
    }
}

func rateLimitMiddleware(rl *rateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr
            
            if !rl.allow(ip) {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

Usage:

```go
limiter := newRateLimiter(100, time.Minute)  // 100 requests per minute
r.Use(rateLimitMiddleware(limiter))
```

If a client makes more than 100 requests in a minute, they get a **429 Too Many Requests** response.

## Authentication Middleware Preview

Authentication is a big topic, but here is a simple token-based auth middleware:

```go
type contextKey string
const userIDKey contextKey = "userID"

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get the Authorization header
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Missing Authorization header", http.StatusUnauthorized)
            return
        }
        
        // Check format: "Bearer <token>"
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid Authorization format", http.StatusUnauthorized)
            return
        }
        
        token := parts[1]
        
        // Validate the token (simplified for now)
        userID, valid := validateToken(token)
        if !valid {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        // Store user ID in context for handlers to use
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        r = r.WithContext(ctx)
        
        next.ServeHTTP(w, r)
    })
}

func validateToken(token string) (int, bool) {
    // In a real app, verify JWT or check database
    // For now, just check if it is not empty
    if token == "secret123" {
        return 1, true
    }
    return 0, false
}

// Helper for handlers to get the authenticated user ID
func GetUserID(r *http.Request) (int, bool) {
    if id, ok := r.Context().Value(userIDKey).(int); ok {
        return id, true
    }
    return 0, false
}
```

## Timing Middleware

Want to know how long each request takes and log slow ones? Here is a timing middleware:

```go
func timingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        recorder := &responseRecorder{ResponseWriter: w, statusCode: 200}
        
        next.ServeHTTP(recorder, r)
        
        duration := time.Since(start)
        
        // Log slow requests
        if duration > 500*time.Millisecond {
            fmt.Printf("SLOW REQUEST: %s %s took %v\n", r.Method, r.URL.Path, duration)
        }
        
        // Add timing header
        w.Header().Set("X-Response-Time", duration.String())
    })
}
```

## Middleware Order Matters

I cannot stress this enough: **the order of middleware matters**. Let me show you why.

If you have:

```go
r.Use(loggingMiddleware)     // 1st
r.Use(authMiddleware)        // 2nd
r.Use(recoveryMiddleware)    // 3rd
```

A request goes: logging -> auth -> recovery -> handler.

If auth middleware panics, recovery catches it (good). But if logging middleware panics, nothing catches it (bad).

If auth rejects a request, logging has already logged it (good). But if you put auth before logging, rejected requests would not be logged (bad for debugging).

The recommended order:

```go
r.Use(recoveryMiddleware)    // 1st: catch all panics
r.Use(requestIDMiddleware)   // 2nd: assign request ID
r.Use(loggingMiddleware)     // 3rd: log everything (with request ID)
r.Use(timingMiddleware)      // 4th: measure timing
r.Use(corsMiddleware)        // 5th: handle CORS
r.Use(rateLimitMiddleware)   // 6th: rate limit
r.Use(authMiddleware)        // 7th: authentication (last before handler)
```

## Real-World Middleware Stack

Here is what a production-ready middleware stack looks like:

```go
func main() {
    r := chi.NewRouter()
    
    // 1. Recover from panics first
    r.Use(middleware.Recoverer)
    
    // 2. Assign request IDs
    r.Use(requestIDMiddleware)
    
    // 3. Log all requests with IDs
    r.Use(loggingMiddleware)
    
    // 4. Measure request timing
    r.Use(timingMiddleware)
    
    // 5. Handle CORS
    r.Use(corsMiddleware)
    
    // 6. Rate limit
    limiter := newRateLimiter(100, time.Minute)
    r.Use(rateLimitMiddleware(limiter))
    
    // Public routes (no auth)
    r.Get("/products", listProducts)
    r.Get("/products/{id}", getProduct)
    
    // Protected routes (auth required)
    r.Group(func(r chi.Router) {
        r.Use(authMiddleware)
        r.Post("/products", createProduct)
        r.Put("/products/{id}", updateProduct)
        r.Delete("/products/{id}", deleteProduct)
    })
    
    http.ListenAndServe(":8080", r)
}
```

This stack handles panics, tracks requests, logs everything, measures performance, handles CORS, limits abuse, and protects sensitive routes. Every real production server needs most of these. And they are all just middleware, pluggable and reusable.
