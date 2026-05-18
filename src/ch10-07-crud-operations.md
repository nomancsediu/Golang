# CRUD Operations in Go

## What is CRUD?

**CRUD** stands for **Create, Read, Update, Delete**. These are the four basic operations you perform on any data in a database. Almost every application is built on CRUD at its core.

When I first started with Go, I wrote database queries directly in my HTTP handlers. It worked, but it was messy. SQL strings were scattered everywhere. Changing the database schema meant updating dozens of files. Testing was impossible without a real database.

The solution is the **Repository pattern**. You define an interface with CRUD methods and implement it with a specific database. Your handlers depend on the interface, not the database.

## The Repository Interface

First, define the interface:

```go
package repository

import "myapp/domain"

type ProductRepository interface {
    Create(product *domain.Product) error
    FindByID(id int64) (*domain.Product, error)
    FindAll() ([]*domain.Product, error)
    Update(product *domain.Product) error
    Delete(id int64) error
}
```

This interface describes what the repository can do, not how it does it. The domain layer uses this interface. The infrastructure layer implements it.

## The Product Entity

Here is the domain model:

```go
package domain

import "time"

type Product struct {
    ID          int64      `json:"id"`
    Name        string     `json:"name"`
    Description string     `json:"description"`
    Price       float64    `json:"price"`
    Stock       int        `json:"stock"`
    Category    string     `json:"category"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   *time.Time `json:"updated_at"`
}
```

## PostgreSQL Implementation

Now implement the repository with PostgreSQL:

```go
package postgres

import (
    "database/sql"
    "fmt"
    "myapp/domain"

    _ "github.com/lib/pq"
)

type ProductRepo struct {
    db *sql.DB
}

func NewProductRepo(db *sql.DB) *ProductRepo {
    return &ProductRepo{db: db}
}
```

## Create

```go
func (r *ProductRepo) Create(product *domain.Product) error {
    query := `
        INSERT INTO products (name, description, price, stock, category)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id, created_at`

    err := r.db.QueryRow(
        query,
        product.Name,
        product.Description,
        product.Price,
        product.Stock,
        product.Category,
    ).Scan(&product.ID, &product.CreatedAt)

    if err != nil {
        return fmt.Errorf("failed to create product: %w", err)
    }

    return nil
}
```

After this function runs, the product struct has its ID and CreatedAt filled in. This is useful because the caller can immediately use the generated ID.

## Read (FindByID)

```go
func (r *ProductRepo) FindByID(id int64) (*domain.Product, error) {
    query := `
        SELECT id, name, description, price, stock, category, created_at, updated_at
        FROM products
        WHERE id = $1`

    var p domain.Product
    err := r.db.QueryRow(query, id).Scan(
        &p.ID,
        &p.Name,
        &p.Description,
        &p.Price,
        &p.Stock,
        &p.Category,
        &p.CreatedAt,
        &p.UpdatedAt,
    )

    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("product with id %d not found", id)
        }
        return nil, fmt.Errorf("failed to find product: %w", err)
    }

    return &p, nil
}
```

Always check for **sql.ErrNoRows** separately. "Not found" is not the same as "database error."

## Read (FindAll)

```go
func (r *ProductRepo) FindAll() ([]*domain.Product, error) {
    query := `
        SELECT id, name, description, price, stock, category, created_at, updated_at
        FROM products
        ORDER BY id`

    rows, err := r.db.Query(query)
    if err != nil {
        return nil, fmt.Errorf("failed to query products: %w", err)
    }
    defer rows.Close()

    products := make([]*domain.Product, 0)
    for rows.Next() {
        var p domain.Product
        err := rows.Scan(
            &p.ID,
            &p.Name,
            &p.Description,
            &p.Price,
            &p.Stock,
            &p.Category,
            &p.CreatedAt,
            &p.UpdatedAt,
        )
        if err != nil {
            return nil, fmt.Errorf("failed to scan product: %w", err)
        }
        products = append(products, &p)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("error iterating products: %w", err)
    }

    return products, nil
}
```

I initialize the slice with `make([]*domain.Product, 0)` instead of just declaring it. This way, if there are no products, the function returns an empty slice instead of nil. This makes JSON encoding consistent: it returns `[]` instead of `null`.

## Update

```go
func (r *ProductRepo) Update(product *domain.Product) error {
    query := `
        UPDATE products
        SET name = $1, description = $2, price = $3, stock = $4,
            category = $5, updated_at = NOW()
        WHERE id = $6
        RETURNING updated_at`

    err := r.db.QueryRow(
        query,
        product.Name,
        product.Description,
        product.Price,
        product.Stock,
        product.Category,
        product.ID,
    ).Scan(&product.UpdatedAt)

    if err != nil {
        if err == sql.ErrNoRows {
            return fmt.Errorf("product with id %d not found", product.ID)
        }
        return fmt.Errorf("failed to update product: %w", err)
    }

    return nil
}
```

Notice I use `RETURNING updated_at` and `NOW()` to automatically set the updated_at timestamp. This ensures the timestamp is always correct, even if the caller forgets to set it.

## Delete

```go
func (r *ProductRepo) Delete(id int64) error {
    query := `DELETE FROM products WHERE id = $1`

    result, err := r.db.Exec(query, id)
    if err != nil {
        return fmt.Errorf("failed to delete product: %w", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("failed to get rows affected: %w", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("product with id %d not found", id)
    }

    return nil
}
```

I use **result.RowsAffected()** to check if any row was actually deleted. If zero rows were affected, the product did not exist.

## Using the Repository

Here is how to wire it all together:

```go
func main() {
    db, err := sql.Open("postgres", "postgres://localhost/myapp?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Create the repository
    repo := postgres.NewProductRepo(db)

    // Create a product
    product := &domain.Product{
        Name:        "Go Book",
        Description: "Learn Go programming",
        Price:       29.99,
        Stock:       100,
        Category:    "Books",
    }
    err = repo.Create(product)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created product with ID: %d\n", product.ID)

    // Find the product
    found, err := repo.FindByID(product.ID)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Found product: %s\n", found.Name)

    // Update the product
    found.Price = 24.99
    err = repo.Update(found)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Updated price to: %.2f\n", found.Price)

    // Delete the product
    err = repo.Delete(found.ID)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Product deleted")
}
```

## Error Handling Summary

Each CRUD operation has its own error patterns:

| Operation | Common Errors | How to Handle |
|-----------|---------------|---------------|
| **Create** | Duplicate key, foreign key violation | Check error codes, return meaningful messages |
| **Read** | No rows found | Check sql.ErrNoRows, return "not found" |
| **Update** | No rows matched | Check RowsAffected, return "not found" |
| **Delete** | No rows matched, foreign key constraint | Check RowsAffected, return appropriate error |

The repository pattern makes all of this consistent. Every operation returns a clear error. Every handler can trust the repository to do the right thing.
