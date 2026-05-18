# Find Operation in Go

## Querying Data from the Database

After inserting data, you need to read it back. Go's `database/sql` package provides two main functions for querying:

- **db.Query()** - Returns multiple rows. Use for SELECT queries that return more than one row.
- **db.QueryRow()** - Returns a single row. Use for SELECT queries that return exactly one row.

## Finding a Single Row with QueryRow

Use `db.QueryRow()` when you expect exactly one result, like finding a user by ID:

```go
func FindUserByID(db *sql.DB, id int64) (*User, error) {
    var user User
    err := db.QueryRow(
        "SELECT id, email, name, created_at FROM users WHERE id = $1",
        id,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.CreatedAt)

    if err != nil {
        if err == sql.ErrNoRows {
            // No user found with this ID
            return nil, fmt.Errorf("user not found")
        }
        // Some other error
        return nil, fmt.Errorf("failed to query user: %w", err)
    }

    return &user, nil
}
```

**sql.ErrNoRows** is a special error that means the query returned no results. Always check for it separately. It is not really an error. It just means the record does not exist.

## Finding Multiple Rows with Query

Use `db.Query()` when you expect multiple results:

```go
func FindAllProducts(db *sql.DB) ([]Product, error) {
    rows, err := db.Query(
        "SELECT id, name, price, stock, category FROM products ORDER BY id",
    )
    if err != nil {
        return nil, fmt.Errorf("failed to query products: %w", err)
    }
    // Always close rows when done
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock, &p.Category)
        if err != nil {
            return nil, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, p)
    }

    // Check for errors that happened during iteration
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("error during row iteration: %w", err)
    }

    return products, nil
}
```

Three important things to remember with `db.Query()`:

1. **Always defer rows.Close()** - If you do not close the rows, the database connection stays open and leaks resources.
2. **Always call rows.Next()** in a loop - This advances to the next row.
3. **Always check rows.Err()** after the loop - This catches errors that happened during iteration.

## Scanning Results into a Struct

The **Scan** function reads column values from the current row into Go variables. The order of variables must match the order of columns in your SELECT:

```go
// The columns in SELECT must match the Scan arguments
err := db.QueryRow(
    "SELECT id, email, name, created_at FROM users WHERE id = $1",
    id,
).Scan(&user.ID, &user.Email, &user.Name, &user.CreatedAt)

// If you SELECT *, the order depends on the table definition
// It is safer to name columns explicitly
err := db.QueryRow(
    "SELECT * FROM users WHERE id = $1",  // risky
    id,
).Scan(&user.ID, &user.Email, &user.Name, &user.PasswordHash, &user.CreatedAt)
```

I always name the columns explicitly. `SELECT *` is fragile. If you add a column to the table, your Scan will break because the column count changes.

## Handling NULL Values

PostgreSQL columns can be NULL. But Go types like `string` and `int` cannot be nil. If a column might be NULL, you need special types:

```go
import "database/sql"

type Product struct {
    ID          int64
    Name        string
    Description string // might be NULL in the database
    Price       float64
    Category    string // might be NULL in the database
}

func FindProductByID(db *sql.DB, id int64) (*Product, error) {
    var p Product
    var description sql.NullString
    var category sql.NullString

    err := db.QueryRow(
        "SELECT id, name, description, price, category FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, &description, &p.Price, &category)

    if err != nil {
        return nil, fmt.Errorf("failed to query product: %w", err)
    }

    // Convert NullString to regular string
    if description.Valid {
        p.Description = description.String
    }
    if category.Valid {
        p.Category = category.String
    }

    return &p, nil
}
```

The **sql.NullString** type has two fields: `String` (the value) and `Valid` (whether the value is not NULL). Similar types exist for other Go types:

- **sql.NullInt64** - For nullable INTEGER columns
- **sql.NullFloat64** - For nullable NUMERIC columns
- **sql.NullBool** - For nullable BOOLEAN columns
- **sql.NullTime** - For nullable TIMESTAMP columns

## Alternative: Use Pointers

Another way to handle NULL values is to use pointers. A nil pointer means NULL:

```go
type Product struct {
    ID          int64
    Name        string
    Description *string  // nil means NULL
    Price       float64
    Category    *string  // nil means NULL
}

func FindProductByID(db *sql.DB, id int64) (*Product, error) {
    var p Product
    err := db.QueryRow(
        "SELECT id, name, description, price, category FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Category)

    if err != nil {
        return nil, fmt.Errorf("failed to query product: %w", err)
    }
    return &p, nil
}
```

This is cleaner than sql.NullString. I prefer pointer types for nullable columns.

## Finding with Filters

Most real queries have filters. Here is how to build a find function with a where clause:

```go
func FindProductsByCategory(db *sql.DB, category string) ([]Product, error) {
    rows, err := db.Query(
        "SELECT id, name, price, stock FROM products WHERE category = $1",
        category,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to query products: %w", err)
    }
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
            return nil, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, p)
    }

    return products, rows.Err()
}
```

## Count Query

Sometimes you just need a count:

```go
func CountProducts(db *sql.DB) (int, error) {
    var count int
    err := db.QueryRow("SELECT COUNT(*) FROM products").Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("failed to count products: %w", err)
    }
    return count, nil
}
```

## Find with Join

For queries that join multiple tables:

```go
func FindOrdersWithItems(db *sql.DB, userID int64) ([]OrderWithItems, error) {
    rows, err := db.Query(
        `SELECT o.id, o.total, o.status, oi.product_id, oi.quantity, oi.price
         FROM orders o
         JOIN order_items oi ON o.id = oi.order_id
         WHERE o.user_id = $1
         ORDER BY o.id`,
        userID,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to query orders: %w", err)
    }
    defer rows.Close()

    var results []OrderWithItems
    for rows.Next() {
        var r OrderWithItems
        if err := rows.Scan(&r.OrderID, &r.Total, &r.Status,
            &r.ProductID, &r.Quantity, &r.Price); err != nil {
            return nil, fmt.Errorf("failed to scan order: %w", err)
        }
        results = append(results, r)
    }

    return results, rows.Err()
}
```

The pattern is always the same: query, defer close, loop with Next, scan each row, check Err. Once you memorize this pattern, database queries in Go become straightforward.
