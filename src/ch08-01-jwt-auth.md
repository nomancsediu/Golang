# JWT Authentication

## What is JWT?

**JWT** stands for **JSON Web Token**. It is a standard way to securely transmit information between a client and a server as a JSON object. Think of it like a digital ID card that the server gives to the client after they log in.

The client carries this token around and shows it with every request. The server checks the token to verify who the client is.

JWT is not encrypted by default. Anyone can read the content. But it is **signed**, which means nobody can tamper with it without the server knowing. The signature is created using a secret key that only the server knows.

## JWT Structure

A JWT has three parts separated by dots:

```
header.payload.signature
```

**Header** - Contains the token type and the signing algorithm:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload** - Contains the claims (data you want to store):
```json
{
  "user_id": 42,
  "email": "john@example.com",
  "exp": 1700000000
}
```

**Signature** - Used to verify the token has not been tampered with:
```
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

The three parts are Base64Url encoded and joined with dots. The result looks like a long random string.

## How JWT Works

The flow is simple:

1. **User logs in** with email and password
2. **Server validates** the credentials
3. **Server creates** a JWT and signs it with a secret key
4. **Server sends** the JWT back to the client
5. **Client stores** the JWT (usually in localStorage or a cookie)
6. **Client sends** the JWT with every request in the Authorization header
7. **Server verifies** the JWT on each request
8. If valid, the server processes the request. If invalid, the server returns **401 Unauthorized**

This is stateless auth. The server does not need to store sessions. All the info is in the token itself.

## Installing the JWT Package

We will use the **golang-jwt** package:

```bash
go get github.com/golang-jwt/jwt/v5
```

## Creating a JWT

Here is how you create a token when a user logs in:

```go
package main

import (
    "fmt"
    "time"
    "github.com/golang-jwt/jwt/v5"
)

// Secret key - in production, load from environment variable
var jwtSecret = []byte("my-super-secret-key")

// Claims that we want to store in the token
type Claims struct {
    UserID int64  `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

// CreateToken generates a JWT for a given user
func CreateToken(userID int64, email string) (string, error) {
    // Set the claims (data inside the token)
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-app",
        },
    }

    // Create the token with claims
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

    // Sign the token with our secret key
    tokenString, err := token.SignedString(jwtSecret)
    if err != nil {
        return "", fmt.Errorf("failed to sign token: %w", err)
    }

    return tokenString, nil
}
```

## Verifying a JWT

Every time a request comes in, we need to verify the token:

```go
// VerifyToken checks if a token is valid and returns the claims
func VerifyToken(tokenString string) (*Claims, error) {
    // Parse the token
    token, err := jwt.ParseWithClaims(tokenString, &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            // Make sure the signing method is what we expect
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v",
                    token.Header["alg"])
            }
            return jwtSecret, nil
        },
    )

    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }

    // Extract the claims
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }

    return claims, nil
}
```

## Complete Login Endpoint

Here is a full login handler that returns a JWT:

```go
package main

import (
    "encoding/json"
    "net/http"
    "golang.org/x/crypto/bcrypt"
)

// User model (simplified)
type User struct {
    ID       int64  `json:"id"`
    Email    string `json:"email"`
    Password string `json:"-"` // never expose password in JSON
}

// Login request
type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

// Login response
type LoginResponse struct {
    Token string `json:"token"`
    User  User   `json:"user"`
}

// LoginHandler handles user login
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // Only accept POST
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    // Parse request body
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Find user by email (in real app, query database)
    user, err := FindUserByEmail(req.Email)
    if err != nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }

    // Compare password hash
    if err := bcrypt.CompareHashAndPassword(
        []byte(user.Password), []byte(req.Password),
    ); err != nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }

    // Create JWT token
    token, err := CreateToken(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to create token", http.StatusInternalServerError)
        return
    }

    // Send response
    resp := LoginResponse{
        Token: token,
        User:  *user,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}
```

## Important JWT Tips

- **Never put sensitive data** in the payload. Anyone can decode a JWT and read it.
- **Always set an expiration**. Tokens that never expire are a security risk.
- **Keep your secret key safe**. Store it in an environment variable, not in your code.
- **Use HTTPS**. Tokens sent over plain HTTP can be intercepted.
- **Use strong secrets**. A weak secret can be brute-forced to forge tokens.

JWT is not perfect. It has trade-offs like any approach. But for most Go APIs, it is the simplest and most practical way to handle authentication.
