# Preflight Request

This is one of those topics that confused me for a long time. I would build an API, test it with curl, and everything worked fine. But when I tried to call it from a browser using JavaScript, I got weird errors about CORS. Then I noticed the browser was sending an extra request I never asked for: an **OPTIONS** request.

That extra request is called a **preflight request**, and understanding it is essential for building APIs that work with browsers.

## What is a Preflight Request?

A preflight request is an **OPTIONS request** that the browser sends automatically before your actual request. It is the browser's way of asking the server, "Hey, is it okay if I send this request?"

Here is the flow:

1. Your JavaScript code wants to send a POST request to an API
2. The browser checks if this is a "simple request" or a "complex request"
3. If it is complex, the browser sends a **preflight OPTIONS request** first
4. The server responds with CORS headers saying what is allowed
5. If the preflight passes, the browser sends the actual POST request

The key thing to understand: **the preflight is sent by the browser, not by your code**. You never write code to send a preflight. The browser does it automatically.

## Why Do Browsers Send Preflight Requests?

It is all about **security**. Specifically, **CORS** (Cross-Origin Resource Sharing).

Imagine you are logged into your bank at `bank.com`. You then visit a malicious site at `evil.com`. Without CORS protection, the JavaScript on `evil.com` could send requests to `bank.com` and steal your data, because your browser automatically includes your bank's cookies.

**CORS** prevents this by requiring the server to explicitly say which origins are allowed to make requests. The preflight request is the browser's way of checking this permission before sending the actual request.

## When is a Preflight Triggered?

Not every request triggers a preflight. Browsers categorize requests into **simple requests** and **complex requests**.

**Simple requests** (no preflight needed):
- GET, HEAD, or POST only
- Content-Type is only `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`
- No custom headers

**Complex requests** (preflight IS triggered):
- Methods like PUT, DELETE, PATCH
- Content-Type is `application/json`
- Custom headers like `Authorization` or `X-Custom-Header`

This is why your API works fine with curl (curl does not enforce CORS) but fails from a browser. When your frontend sends `Content-Type: application/json`, the browser automatically sends a preflight first.

## What the Preflight Looks Like

Here is what a preflight request looks like:

```
OPTIONS /products HTTP/1.1
Host: api.example.com
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

And here is what the server response should look like:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

Let me explain each header:

- **`Access-Control-Allow-Origin`** - Which origins are allowed. Use `*` for public APIs, or a specific origin for private ones.
- **`Access-Control-Allow-Methods`** - Which HTTP methods are allowed.
- **`Access-Control-Allow-Headers`** - Which custom headers are allowed.
- **`Access-Control-Max-Age`** - How long the browser can cache this preflight result (in seconds). Saves redundant preflight requests.

## Handling Preflight in Go

If you do not handle preflight requests, your frontend will get errors. Here is how to handle them:

```go
func enableCORS(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        // Handle preflight request
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)  // 204
            return
        }
        
        // Not a preflight, pass to the next handler
        next(w, r)
    }
}
```

Wait, that looks like middleware. That is because it IS middleware. We will cover middleware properly later, but for now, think of it as a function that wraps your handler and adds CORS handling.

## Using the CORS Handler

Here is a complete example with CORS support:

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

var products = []Product{
    {ID: 1, Name: "Go Book", Price: 39.99},
}

func withCORS(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "86400")
        
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        
        next(w, r)
    }
}

func productsHandler(w http.ResponseWriter, r *http.Request) {
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
    }
}

func main() {
    http.HandleFunc("/products", withCORS(productsHandler))
    http.ListenAndServe(":8080", nil)
}
```

Now when a browser sends a preflight OPTIONS request, your server responds with the right CORS headers. The browser then sends the actual request.

## Common Gotchas

**Returning 200 instead of 204 for preflight** - Some browsers accept this, but the correct response for a preflight is **204 No Content**. It makes sense: the preflight is just a permission check, there is no content to return.

**Setting CORS headers only on preflight** - You need to set CORS headers on ALL responses, not just the preflight. The browser checks CORS headers on the actual response too.

**Using `Access-Control-Allow-Origin: *` with credentials** - If your frontend sends cookies or auth headers, you cannot use `*` for the origin. You must specify the exact origin.

Preflight requests can be frustrating when you first encounter them, but they make sense once you understand the security model. The browser is just being careful, and your server needs to cooperate.
