# Query Parameters in Go

## Reading Query Parameters

**Query parameters** are the key-value pairs that appear after the `?` in a URL:

```
GET /api/products?category=Books&sort=price&order=asc&page=1&limit=20
```

In Go, you read query parameters using `r.URL.Query()`:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()

    category := query.Get("category")    // "Books"
    sort := query.Get("sort")            // "price"
    order := query.Get("order")          // "asc"
    page := query.Get("page")            // "1"
    limit := query.Get("limit")          // "20"

    // Note: all values are strings. Convert as needed.
}
```

`query.Get()` returns an empty string if the parameter is not present. It never returns an error.

## Common Query Parameters

Here are the parameters I use most often in APIs:

| Parameter | Purpose | Example |
|-----------|---------|---------|
| **page** | Pagination page number | `?page=2` |
| **limit** | Items per page | `?limit=20` |
| **sort** | Sort field | `?sort=price` |
| **order** | Sort direction | `?order=asc` or `?order=desc` |
| **search** | Full-text search | `?search=go+book` |
| **filter** | Filter by field | `?category=Books` |

## Parsing and Validating Query Parameters

You should always validate and sanitize query parameters:

```go
type ProductFilter struct {
    Page     int
    Limit    int
    Sort     string
    Order    string
    Category string
    Search   string
    MinPrice float64
    MaxPrice float64
}

func ParseProductFilter(r *http.Request) ProductFilter {
    query := r.URL.Query()

    f := ProductFilter{
        Page:     parseInt(query.Get("page"), 1),
        Limit:    parseInt(query.Get("limit"), 20),
        Sort:     parseSort(query.Get("sort")),
        Order:    parseOrder(query.Get("order")),
        Category: query.Get("category"),
        Search:   query.Get("search"),
        MinPrice: parseFloat(query.Get("min_price"), 0),
        MaxPrice: parseFloat(query.Get("max_price"), 0),
    }

    // Apply limits
    if f.Limit < 1 {
        f.Limit = 20
    }
    if f.Limit > 100 {
        f.Limit = 100
    }
    if f.Page < 1 {
        f.Page = 1
    }

    return f
}

func parseInt(s string, defaultVal int) int {
    n, err := strconv.Atoi(s)
    if err != nil {
        return defaultVal
    }
    return n
}

func parseFloat(s string, defaultVal float64) float64 {
    f, err := strconv.ParseFloat(s, 64)
    if err != nil {
        return defaultVal
    }
    return f
}

func parseSort(s string) string {
    allowed := map[string]bool{
        "price": true, "name": true, "created_at": true, "stock": true,
    }
    if allowed[s] {
        return s
    }
    return "id" // default sort
}

func parseOrder(s string) string {
    if s == "desc" {
        return "DESC"
    }
    return "ASC" // default order
}
```

Why validate sort and order? To prevent **SQL injection**. Never trust user input for column names. Only allow whitelisted values.

## Building Dynamic Queries

The tricky part is building SQL queries from filters. You need to dynamically add WHERE clauses based on which parameters are present:

```go
func (r *ProductRepo) FindFiltered(f ProductFilter) ([]*domain.Product, int64, error) {
    // Build the query dynamically
    where := []string{}
    args := []interface{}{}
    argPos := 1

    if f.Category != "" {
        where = append(where, fmt.Sprintf("category = $%d", argPos))
        args = append(args, f.Category)
        argPos++
    }

    if f.Search != "" {
        where = append(where, fmt.Sprintf("name ILIKE $%d", argPos))
        args = append(args, "%"+f.Search+"%")
        argPos++
    }

    if f.MinPrice > 0 {
        where = append(where, fmt.Sprintf("price >= $%d", argPos))
        args = append(args, f.MinPrice)
        argPos++
    }

    if f.MaxPrice > 0 {
        where = append(where, fmt.Sprintf("price <= $%d", argPos))
        args = append(args, f.MaxPrice)
        argPos++
    }

    // Build WHERE clause
    whereClause := ""
    if len(where) > 0 {
        whereClause = "WHERE " + strings.Join(where, " AND ")
    }

    // Count total
    countQuery := fmt.Sprintf("SELECT COUNT(*) FROM products %s", whereClause)
    var total int64
    err := r.db.QueryRow(countQuery, args...).Scan(&total)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to count products: %w", err)
    }

    // Build sort clause (already validated)
    sortClause := fmt.Sprintf("ORDER BY %s %s", f.Sort, f.Order)

    // Add pagination
    limit := f.Limit
    offset := (f.Page - 1) * f.Limit
    args = append(args, limit, offset)

    dataQuery := fmt.Sprintf(
        "SELECT id, name, price, stock, category FROM products %s %s LIMIT $%d OFFSET $%d",
        whereClause, sortClause, argPos, argPos+1,
    )

    rows, err := r.db.Query(dataQuery, args...)
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

The key insight: **always use parameterized queries for values** ($1, $2, etc.) but you can safely concatenate column names and SQL keywords if they are validated against a whitelist.

## Multiple Values for the Same Parameter

Sometimes a parameter can have multiple values:

```
GET /api/products?category=Books&category=Electronics
```

Use `query["category"]` (a slice) instead of `query.Get("category")` (first value only):

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()

    // Get all values for a parameter
    categories := query["category"] // ["Books", "Electronics"]

    // Or check if a parameter has multiple values
    if len(categories) > 1 {
        // Handle multiple categories
        // WHERE category IN ('Books', 'Electronics')
    }
}
```

## Default Values

Always provide sensible defaults so the API works even without query parameters:

```go
func ParseProductFilter(r *http.Request) ProductFilter {
    query := r.URL.Query()

    return ProductFilter{
        Page:     defaultInt(query.Get("page"), 1),
        Limit:    defaultInt(query.Get("limit"), 20),
        Sort:     defaultSort(query.Get("sort"), "id"),
        Order:    defaultOrder(query.Get("order"), "ASC"),
        Category: query.Get("category"), // empty means no filter
        Search:   query.Get("search"),   // empty means no search
    }
}
```

## Complete Handler Example

Here is a complete product listing handler with all query parameters:

```go
func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    filter := ParseProductFilter(r)

    products, total, err := h.repo.FindFiltered(filter)
    if err != nil {
        respondJSON(w, http.StatusInternalServerError, map[string]string{
            "error": "Failed to fetch products",
        })
        return
    }

    pg := pagination.NewResponse(pagination.NewRequest(filter.Page, filter.Limit), total)

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "data":       products,
        "pagination": pg,
        "filters":    filter,
    })
}
```

Query parameters make your API flexible. Instead of building separate endpoints for every filter combination, one endpoint handles them all. The client decides what data they need.
