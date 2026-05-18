# Pagination in Go APIs

## Why Pagination Matters

Imagine you have 1 million products in your database. A client requests `GET /api/products`. Without pagination, your server loads all 1 million rows into memory, converts them to JSON, and sends a massive response. This will:

- **Use too much memory** on the server
- **Take too long** to respond
- **Use too much bandwidth** transferring the data
- **Crash the client** trying to parse a huge JSON response

Pagination solves this by returning a small subset of results at a time. Instead of 1 million products, you return 20 products per page. The client requests page 1, page 2, page 3, and so on.

## Basic Pagination: Limit and Offset

The simplest pagination uses two parameters:

- **limit** - How many items per page (e.g., 20)
- **offset** - How many items to skip (e.g., 0 for page 1, 20 for page 2)

In SQL, this maps directly to `LIMIT` and `OFFSET`:

```sql
-- Page 1: items 1-20
SELECT * FROM products LIMIT 20 OFFSET 0;

-- Page 2: items 21-40
SELECT * FROM products LIMIT 20 OFFSET 20;

-- Page 3: items 41-60
SELECT * FROM products LIMIT 20 OFFSET 40;
```

## Pagination Request Format

Clients typically send pagination as query parameters:

```
GET /api/products?page=1&limit=20
```

Or sometimes:

```
GET /api/products?offset=0&limit=20
```

I prefer the **page/limit** format because it is more intuitive. Nobody thinks in terms of offsets. Everyone thinks in terms of pages.

## Pagination Response Format

A good pagination response includes the data plus metadata about the pagination:

```json
{
    "data": [
        {"id": 1, "name": "Product A", "price": 29.99},
        {"id": 2, "name": "Product B", "price": 19.99}
    ],
    "pagination": {
        "page": 1,
        "limit": 20,
        "total": 150,
        "total_pages": 8
    }
}
```

The metadata tells the client:
- **page** - Current page number
- **limit** - Items per page
- **total** - Total number of items
- **total_pages** - Total number of pages

## Building a Generic Pagination Function

Here is a reusable pagination structure:

```go
package pagination

import "math"

type Request struct {
    Page  int `json:"page"`
    Limit int `json:"limit"`
}

type Response struct {
    Page       int   `json:"page"`
    Limit      int   `json:"limit"`
    Total      int64 `json:"total"`
    TotalPages int   `json:"total_pages"`
}

func NewRequest(page, limit int) Request {
    // Apply defaults
    if page < 1 {
        page = 1
    }
    if limit < 1 {
        limit = 20
    }
    if limit > 100 {
        limit = 100 // Maximum limit to prevent abuse
    }
    return Request{Page: page, Limit: limit}
}

func (r Request) Offset() int {
    return (r.Page - 1) * r.Limit
}

func NewResponse(req Request, total int64) Response {
    totalPages := int(math.Ceil(float64(total) / float64(req.Limit)))
    if totalPages < 1 {
        totalPages = 1
    }
    return Response{
        Page:       req.Page,
        Limit:      req.Limit,
        Total:      total,
        TotalPages: totalPages,
    }
}
```

## Using Pagination in a Repository

```go
func (r *ProductRepo) FindAll(p pg.Request) ([]*domain.Product, int64, error) {
    // Count total products
    var total int64
    err := r.db.QueryRow("SELECT COUNT(*) FROM products").Scan(&total)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to count products: %w", err)
    }

    // Query with pagination
    rows, err := r.db.Query(
        "SELECT id, name, price, stock, category FROM products ORDER BY id LIMIT $1 OFFSET $2",
        p.Limit, p.Offset(),
    )
    if err != nil {
        return nil, 0, fmt.Errorf("failed to query products: %w", err)
    }
    defer rows.Close()

    products := make([]*domain.Product, 0)
    for rows.Next() {
        var p domain.Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock, &p.Category); err != nil {
            return nil, 0, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, &p)
    }

    return products, total, rows.Err()
}
```

## Using Pagination in a Handler

```go
func (h *ProductHandler) ListProducts(w http.ResponseWriter, r *http.Request) {
    // Parse pagination from query parameters
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

    // Create pagination request with defaults
    pgReq := pagination.NewRequest(page, limit)

    // Query the repository
    products, total, err := h.repo.FindAll(pgReq)
    if err != nil {
        http.Error(w, "Failed to fetch products", http.StatusInternalServerError)
        return
    }

    // Build pagination response
    pgResp := pagination.NewResponse(pgReq, total)

    // Send response
    respondJSON(w, http.StatusOK, map[string]interface{}{
        "data":       products,
        "pagination": pgResp,
    })
}
```

## Complete Paginated API Response

When you call `GET /api/products?page=2&limit=20`, you get:

```json
{
    "data": [
        {"id": 21, "name": "Product U", "price": 15.99, "stock": 30, "category": "Electronics"},
        {"id": 22, "name": "Product V", "price": 45.99, "stock": 15, "category": "Books"}
    ],
    "pagination": {
        "page": 2,
        "limit": 20,
        "total": 150,
        "total_pages": 8
    }
}
```

## Pagination with Filters

You can combine pagination with other query parameters like filters and sorting:

```
GET /api/products?page=1&limit=20&category=Books&sort=price&order=asc
```

```go
func (h *ProductHandler) ListProducts(w http.ResponseWriter, r *http.Request) {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    category := r.URL.Query().Get("category")
    sort := r.URL.Query().Get("sort")

    pgReq := pagination.NewRequest(page, limit)
    products, total, err := h.repo.FindAllFiltered(pgReq, category, sort)
    // ...
}
```

## Pagination Best Practices

- **Always set a maximum limit**. A client requesting `?limit=999999` should be capped at 100 or whatever your max is.
- **Always set sensible defaults**. If the client does not specify page or limit, use page=1 and limit=20.
- **Include total count**. Clients need this to show "Page 2 of 8" in the UI.
- **Use consistent ordering**. Always include ORDER BY. Without it, pagination results are not deterministic.
- **Consider cursor-based pagination** for very large datasets. Offset pagination gets slow when the offset is large.

Pagination is one of those things that seems simple but has many edge cases. Start with offset pagination. It works for most use cases. Switch to cursor-based pagination only when you need it.
