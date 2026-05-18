# Database Data Types

## Mapping PostgreSQL Types to Go Types

One of the trickiest parts of working with databases in Go is matching PostgreSQL types to Go types. Here is a complete mapping:

| PostgreSQL Type | Go Type | Notes |
|-----------------|---------|-------|
| **VARCHAR(n)** | string | Simple mapping |
| **TEXT** | string | Simple mapping |
| **CHAR(n)** | string | Padded with spaces |
| **INTEGER / INT** | int | 32-bit integer |
| **BIGINT** | int64 | 64-bit integer |
| **SMALLINT** | int16 | 16-bit integer |
| **SERIAL** | int | Auto-incrementing |
| **BIGSERIAL** | int64 | Auto-incrementing |
| **BOOLEAN** | bool | Simple mapping |
| **NUMERIC(p,s)** | float64 | Loss of precision possible |
| **REAL** | float32 | 32-bit float |
| **DOUBLE PRECISION** | float64 | 64-bit float |
| **TIMESTAMP** | time.Time | Date and time |
| **DATE** | time.Time | Date only, time is zero |
| **UUID** | string | Scan as string, parse if needed |
| **JSON / JSONB** | []byte | Scan as bytes, unmarshal |
| **BYTEA** | []byte | Binary data |

## Handling Dates and Timestamps

PostgreSQL TIMESTAMP maps to Go's **time.Time**:

```go
type Event struct {
    ID        int64
    Name      string
    CreatedAt time.Time
    UpdatedAt time.Time
}

func FindEvent(db *sql.DB, id int64) (*Event, error) {
    var e Event
    err := db.QueryRow(
        "SELECT id, name, created_at, updated_at FROM events WHERE id = $1",
        id,
    ).Scan(&e.ID, &e.Name, &e.CreatedAt, &e.UpdatedAt)

    if err != nil {
        return nil, fmt.Errorf("failed to find event: %w", err)
    }
    return &e, nil
}
```

Go automatically parses the PostgreSQL timestamp format. You do not need to do any manual parsing.

For nullable timestamps, use a pointer:

```go
type Event struct {
    ID        int64
    Name      string
    CreatedAt time.Time
    DeletedAt *time.Time // nil means not deleted
}
```

## Handling JSON and JSONB

PostgreSQL JSONB is great for storing flexible data. In Go, you scan it as **[]byte** and then unmarshal:

```go
type Product struct {
    ID       int64
    Name     string
    Metadata map[string]interface{} // JSONB column
}

func FindProduct(db *sql.DB, id int64) (*Product, error) {
    var p Product
    var metadataBytes []byte

    err := db.QueryRow(
        "SELECT id, name, metadata FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, &metadataBytes)

    if err != nil {
        return nil, fmt.Errorf("failed to find product: %w", err)
    }

    // Unmarshal the JSONB data
    if len(metadataBytes) > 0 {
        if err := json.Unmarshal(metadataBytes, &p.Metadata); err != nil {
            return nil, fmt.Errorf("failed to unmarshal metadata: %w", err)
        }
    }

    return &p, nil
}
```

For typed JSON data, use a struct instead of a map:

```go
type ProductMetadata struct {
    Brand  string   `json:"brand"`
    Tags   []string `json:"tags"`
    Rating float64  `json:"rating"`
}

type Product struct {
    ID       int64
    Name     string
    Metadata ProductMetadata
}
```

## Handling UUIDs

PostgreSQL has a native UUID type. The simplest approach in Go is to scan it as a **string**:

```go
import "github.com/google/uuid"

type User struct {
    ID    uuid.UUID
    Email string
}

func FindUser(db *sql.DB, id string) (*User, error) {
    var u User
    var idStr string

    err := db.QueryRow(
        "SELECT id, email FROM users WHERE id = $1",
        id,
    ).Scan(&idStr, &u.Email)

    if err != nil {
        return nil, fmt.Errorf("failed to find user: %w", err)
    }

    u.ID, err = uuid.Parse(idStr)
    if err != nil {
        return nil, fmt.Errorf("failed to parse UUID: %w", err)
    }

    return &u, nil
}
```

For a more efficient approach, you can implement a custom scanner:

```go
// Custom type that implements sql.Scanner and driver.Valuer
type UUID uuid.UUID

func (u *UUID) Scan(value interface{}) error {
    if value == nil {
        return nil
    }
    bytes, ok := value.([]byte)
    if !ok {
        return fmt.Errorf("failed to scan UUID: %v", value)
    }
    parsed, err := uuid.Parse(string(bytes))
    if err != nil {
        return err
    }
    *u = UUID(parsed)
    return nil
}

func (u UUID) Value() (driver.Value, error) {
    return uuid.UUID(u).String(), nil
}
```

## Handling Arrays

PostgreSQL supports array types like `INTEGER[]` and `TEXT[]`. In Go, you need to handle them carefully:

```go
// Using lib/pq for array support
import "github.com/lib/pq"

type Product struct {
    ID    int64
    Name  string
    Tags  []string // PostgreSQL TEXT[]
}

func FindProduct(db *sql.DB, id int64) (*Product, error) {
    var p Product

    err := db.QueryRow(
        "SELECT id, name, tags FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, pq.Array(&p.Tags))

    if err != nil {
        return nil, fmt.Errorf("failed to find product: %w", err)
    }

    return &p, nil
}
```

The **pq.Array** helper converts between Go slices and PostgreSQL arrays. Without it, scanning arrays would fail.

## Custom Scanner and Valuer

For complex types, you can create custom **sql.Scanner** and **driver.Valuer** implementations:

```go
// Money type stored as NUMERIC in PostgreSQL
type Money struct {
    Amount   float64
    Currency string
}

// Scanner: How to read from the database
func (m *Money) Scan(value interface{}) error {
    if value == nil {
        return nil
    }
    // PostgreSQL returns NUMERIC as a string
    str, ok := value.(string)
    if !ok {
        return fmt.Errorf("cannot scan %T into Money", value)
    }
    amount, err := strconv.ParseFloat(str, 64)
    if err != nil {
        return err
    }
    m.Amount = amount
    m.Currency = "USD" // default currency
    return nil
}

// Valuer: How to write to the database
func (m Money) Value() (driver.Value, error) {
    return fmt.Sprintf("%.2f", m.Amount), nil
}
```

With these methods, you can use `Money` directly in Scan:

```go
var price Money
err := db.QueryRow(
    "SELECT price FROM products WHERE id = $1", 1,
).Scan(&price)
```

## Nullable Type Summary

Here is a quick reference for handling NULL in Go:

| Approach | Pros | Cons |
|----------|------|------|
| **sql.NullString** | Standard library | Verbose, need to check .Valid |
| **Pointer types (\*string)** | Clean, idiomatic | Nil panics if not checked |
| **Custom types** | Reusable, clean | More code to write |

I prefer **pointer types** for most cases. They are the most Go-like way to handle nullable values. Use sql.NullX types when you need to distinguish between "not set" and "null from database."
