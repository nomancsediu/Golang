# Offset Pagination

## How Offset Pagination Works

**Offset pagination** is the most common way to paginate database results. It uses two values:

- **limit** - How many rows to return
- **offset** - How many rows to skip

The SQL is straightforward:

```sql
-- Page 1: first 20 rows
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;

-- Page 2: next 20 rows
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 20;

-- Page 3
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;

-- Generic formula
SELECT * FROM products ORDER BY id
LIMIT {limit} OFFSET {(page - 1) * limit};
```

The offset is calculated as `(page - 1) * limit`. Page 1 has offset 0, page 2 has offset 20, page 3 has offset 40, and so on.

## SQL Syntax Variations

Different databases use different syntax for pagination:

```sql
-- PostgreSQL and MySQL
SELECT * FROM products LIMIT 20 OFFSET 40;

-- PostgreSQL also supports this syntax
SELECT * FROM products OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;

-- SQL Server
SELECT * FROM products
ORDER BY id
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;
```

In Go with PostgreSQL, I use `LIMIT $1 OFFSET $2` because it is the simplest.

## Counting Total Rows

To calculate total pages, you need the total row count. Run a separate COUNT query:

```go
func (r *ProductRepo) FindPaginated(page, limit int) ([]*Product, int64, error) {
    // Count total rows
    var total int64
    err := r.db.QueryRow("SELECT COUNT(*) FROM products").Scan(&total)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to count products: %w", err)
    }

    // Calculate offset
    offset := (page - 1) * limit

    // Query with pagination
    rows, err := r.db.Query(
        "SELECT id, name, price, stock FROM products ORDER BY id LIMIT $1 OFFSET $2",
        limit, offset,
    )
    if err != nil {
        return nil, 0, fmt.Errorf("failed to query products: %w", err)
    }
    defer rows.Close()

    products := make([]*Product, 0)
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
            return nil, 0, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, &p)
    }

    return products, total, rows.Err()
}
```

## Building the Pagination Metadata

Calculate the pagination metadata from the total count:

```go
type PaginationMeta struct {
    CurrentPage int   `json:"current_page"`
    PerPage     int   `json:"per_page"`
    TotalItems  int64 `json:"total_items"`
    TotalPages  int   `json:"total_pages"`
    HasNext     bool  `json:"has_next"`
    HasPrev     bool  `json:"has_prev"`
}

func NewPaginationMeta(page, limit int, total int64) PaginationMeta {
    totalPages := int(math.Ceil(float64(total) / float64(limit)))
    if totalPages < 1 {
        totalPages = 1
    }

    return PaginationMeta{
        CurrentPage: page,
        PerPage:     limit,
        TotalItems:  total,
        TotalPages:  totalPages,
        HasNext:     page < totalPages,
        HasPrev:     page > 1,
    }
}
```

The **HasNext** and **HasPrev** fields are helpful for rendering pagination controls in the UI. Instead of calculating them, the client just checks these booleans.

## Offset Pagination with Filters

Combining offset pagination with filters requires care. The COUNT query and the data query must use the same WHERE clause:

```go
func (r *ProductRepo) FindPaginatedFiltered(page, limit int, category string) ([]*Product, int64, error) {
    var whereClause string
    var args []interface{}

    if category != "" {
        whereClause = "WHERE category = $1"
        args = append(args, category)
    }

    // Count with filter
    countQuery := fmt.Sprintf("SELECT COUNT(*) FROM products %s", whereClause)
    var total int64
    err := r.db.QueryRow(countQuery, args...).Scan(&total)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to count products: %w", err)
    }

    // Query with filter and pagination
    offset := (page - 1) * limit
    dataQuery := fmt.Sprintf(
        "SELECT id, name, price, stock FROM products %s ORDER BY id LIMIT $%d OFFSET $%d",
        whereClause, len(args)+1, len(args)+2,
    )
    queryArgs := append(args, limit, offset)

    rows, err := r.db.Query(dataQuery, queryArgs...)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to query products: %w", err)
    }
    defer rows.Close()

    products := make([]*Product, 0)
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
            return nil, 0, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, &p)
    }

    return products, total, rows.Err()
}
```

## The Problem with Offset

Offset pagination has two main problems:

**1. Slow for large offsets**

When you ask for `OFFSET 1000000`, the database still has to scan through all 1 million rows before returning the results. It just discards the first million rows. This gets slower as the offset grows.

```sql
-- Fast: skipping 0 rows
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;        -- ~1ms

-- Slow: skipping 1 million rows
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 1000000;   -- ~500ms
```

**2. Inconsistent with new data**

If a new product is inserted while a user is browsing, the results can shift. The user might see the same product twice or miss a product entirely.

For example: the user views page 1 and sees products 1-20. Someone inserts a new product at position 5. When the user goes to page 2, product 21 is now in position 20, and the user sees it again.

## When Offset Pagination is Fine

Despite its limitations, offset pagination is fine for most use cases:

- **Small datasets** (under 100,000 rows) - The performance difference is negligible
- **Simple browsing** - When exact consistency is not critical
- **Admin dashboards** - When users just need to browse through data
- **Search results** - When the total count is already approximate

Do not over-engineer. Start with offset pagination. Switch to cursor-based pagination only when you actually hit performance problems.

## Cursor-Based Pagination (Alternative)

For very large datasets, **cursor-based pagination** is better. Instead of skipping rows, you use a cursor (usually the last ID) to find the next set of results:

```sql
-- First page
SELECT * FROM products ORDER BY id LIMIT 20;

-- Next page: use the last ID from the previous page
SELECT * FROM products WHERE id > 20 ORDER BY id LIMIT 20;
```

This is faster because the database uses the index on `id` instead of scanning and discarding rows. But it has trade-offs: no random page access, no total count, and more complex implementation.

## My Experience

I have used offset pagination for every project so far and it has worked fine. The datasets were under 500,000 rows and the performance was acceptable. I only ran into the large-offset problem once, with a table that had 10 million rows. For that project, I switched to cursor-based pagination for the most common queries.

The key takeaway: **use offset pagination by default, optimize when needed**. Premature optimization is wasteful. Solve the problem you actually have, not the problem you might have someday.
