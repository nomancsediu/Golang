# Create Operation in Go

## Inserting Data with db.Exec()

The **Create** operation inserts new rows into a database table. In Go, you use `db.Exec()` to execute INSERT statements:

```go
result, err := db.Exec(
    "INSERT INTO products (name, price, stock) VALUES ($1, $2, $3)",
    "Go Book", 29.99, 100,
)
```

The `db.Exec()` function returns a `sql.Result` and an error. The result gives you information about what happened, like how many rows were affected.

## Parameterized Queries

Notice the `$1, $2, $3` placeholders. These are **parameterized queries**. Instead of putting values directly into the SQL string, you use placeholders and pass the values separately.

This is critical for **security**. Parameterized queries prevent **SQL injection**, where an attacker tricks your application into running malicious SQL:

```go
// DANGEROUS: Never do this
query := fmt.Sprintf("INSERT INTO users (name) VALUES ('%s')", userName)
db.Exec(query) // If userName is "'); DROP TABLE users; --", your table is gone

// SAFE: Always use parameterized queries
db.Exec("INSERT INTO users (name) VALUES ($1)", userName)
```

The database driver handles escaping automatically. No matter what the user inputs, it is treated as data, not as SQL code.

## Getting the Last Inserted ID

After inserting a row, you often want to know the ID that was assigned. In PostgreSQL, you use the **RETURNING** clause:

```go
var id int64
err := db.QueryRow(
    "INSERT INTO products (name, price, stock) VALUES ($1, $2, $3) RETURNING id",
    "Go Book", 29.99, 100,
).Scan(&id)

if err != nil {
    log.Fatalf("Failed to insert product: %v", err)
}
fmt.Printf("Inserted product with ID: %d\n", id)
```

Note that we use `db.QueryRow()` instead of `db.Exec()` here because we are returning a value. `QueryRow()` executes the query and returns a single row that you can scan.

## Creating a Product

Here is a complete function to create a product:

```go
package main

import (
    "database/sql"
    "fmt"
    "time"
)

type Product struct {
    ID          int64
    Name        string
    Description string
    Price       float64
    Stock       int
    Category    string
    CreatedAt   time.Time
}

func CreateProduct(db *sql.DB, product *Product) error {
    err := db.QueryRow(
        `INSERT INTO products (name, description, price, stock, category)
         VALUES ($1, $2, $3, $4, $5)
         RETURNING id, created_at`,
        product.Name, product.Description, product.Price,
        product.Stock, product.Category,
    ).Scan(&product.ID, &product.CreatedAt)

    if err != nil {
        return fmt.Errorf("failed to create product: %w", err)
    }

    return nil
}

// Usage
func main() {
    db, _ := sql.Open("postgres", "postgres://localhost/myapp?sslmode=disable")
    defer db.Close()

    product := &Product{
        Name:        "The Go Programming Language",
        Description: "A comprehensive guide to Go",
        Price:       34.99,
        Stock:       50,
        Category:    "Books",
    }

    err := CreateProduct(db, product)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    fmt.Printf("Created product: ID=%d, Name=%s\n", product.ID, product.Name)
}
```

Notice that after the function runs, the `product` struct has its `ID` and `CreatedAt` fields populated. This is because `Scan` writes the RETURNING values back into the struct.

## Creating a User with Hashed Password

When creating a user, you should **never store plain-text passwords**. Always hash them first:

```go
import (
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID           int64
    Email        string
    Name         string
    PasswordHash string
    CreatedAt    time.Time
}

func CreateUser(db *sql.DB, email, name, password string) (*User, error) {
    // Hash the password with bcrypt
    hashedPassword, err := bcrypt.GenerateFromPassword(
        []byte(password), bcrypt.DefaultCost,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to hash password: %w", err)
    }

    // Insert the user
    user := &User{
        Email:        email,
        Name:         name,
        PasswordHash: string(hashedPassword),
    }

    err = db.QueryRow(
        `INSERT INTO users (email, name, password_hash)
         VALUES ($1, $2, $3)
         RETURNING id, created_at`,
        user.Email, user.Name, user.PasswordHash,
    ).Scan(&user.ID, &user.CreatedAt)

    if err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    return user, nil
}
```

**bcrypt** is the standard for password hashing in Go. It automatically adds a random salt and handles key stretching. Never use MD5, SHA1, or any fast hash for passwords.

## Error Handling for Create Operations

Common errors you will encounter:

```go
func CreateProduct(db *sql.DB, product *Product) error {
    err := db.QueryRow(
        `INSERT INTO products (name, price, stock, category)
         VALUES ($1, $2, $3, $4)
         RETURNING id, created_at`,
        product.Name, product.Price, product.Stock, product.Category,
    ).Scan(&product.ID, &product.CreatedAt)

    if err != nil {
        // Check for specific PostgreSQL errors
        if isDuplicateKeyError(err) {
            return fmt.Errorf("product with this name already exists")
        }
        if isForeignKeyError(err) {
            return fmt.Errorf("referenced category does not exist")
        }
        return fmt.Errorf("failed to create product: %w", err)
    }

    return nil
}

// Helper to check for duplicate key errors
func isDuplicateKeyError(err error) bool {
    // PostgreSQL error code 23505 = unique_violation
    return strings.Contains(err.Error(), "23505")
}
```

## Batch Insert

When you need to insert multiple rows, do not execute one INSERT per row. Use a batch approach:

```go
func CreateProducts(db *sql.DB, products []Product) error {
    // Start a transaction
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("failed to begin transaction: %w", err)
    }
    // Rollback on error
    defer tx.Rollback()

    // Prepare the statement once
    stmt, err := tx.Prepare(
        `INSERT INTO products (name, price, stock, category)
         VALUES ($1, $2, $3, $4)`,
    )
    if err != nil {
        return fmt.Errorf("failed to prepare statement: %w", err)
    }
    defer stmt.Close()

    // Execute for each product
    for _, p := range products {
        _, err := stmt.Exec(p.Name, p.Price, p.Stock, p.Category)
        if err != nil {
            return fmt.Errorf("failed to insert product %s: %w", p.Name, err)
        }
    }

    // Commit the transaction
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("failed to commit transaction: %w", err)
    }

    return nil
}
```

Using a **transaction** means all inserts succeed or none do. If the fifth insert fails, the first four are rolled back. No partial data.
