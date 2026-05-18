# Database Migrations in Go

## What Are Migrations?

**Database migrations** are version control for your database schema. Just like you use Git to track changes in your code, migrations track changes in your database structure.

Without migrations, you have a problem. Your code is version controlled, but your database schema is not. How do you add a new column? You run an ALTER TABLE command manually. But what if another developer on your team does not know about it? Their code breaks because their database does not have the new column.

Migrations solve this by storing every schema change as a file. Each file has an **up** migration (apply the change) and a **down** migration (undo the change). Anyone can run the migrations to bring their database to the latest state.

## Using golang-migrate

The most popular migration tool for Go is **golang-migrate**:

```bash
# Install the CLI tool
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

## Creating Migration Files

Each migration has two files: one for up and one for down:

```bash
# Create a new migration
migrate create -ext sql -dir migrations -seq create_users_table
```

This creates two files in the `migrations` directory:

```
migrations/
├── 000001_create_users_table.up.sql
└── 000001_create_users_table.down.sql
```

## Writing Migrations

The **up** migration applies the change:

```sql
-- 000001_create_users_table.up.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The **down** migration undoes the change:

```sql
-- 000001_create_users_table.down.sql
DROP TABLE IF EXISTS users;
```

Let us create more migrations for our ecommerce app:

```sql
-- 000002_create_products_table.up.sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```sql
-- 000002_create_products_table.down.sql
DROP TABLE IF EXISTS products;
```

```sql
-- 000003_create_orders_table.up.sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price NUMERIC(10, 2) NOT NULL,
    CONSTRAINT unique_order_product UNIQUE (order_id, product_id)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

```sql
-- 000003_create_orders_table.down.sql
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
```

Note: drop tables in reverse order of creation to respect foreign key constraints.

## Running Migrations from CLI

```bash
# Run all pending migrations (up)
migrate -path migrations -database "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable" up

# Rollback the last migration (down by 1)
migrate -path migrations -database "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable" down 1

# Go to a specific version
migrate -path migrations -database "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable" goto 2

# Check current version
migrate -path migrations -database "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable" version

# Force a version (use when migration is dirty)
migrate -path migrations -database "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable" force 2
```

## Running Migrations from Go Code

You can also run migrations programmatically in your Go application:

```go
package database

import (
    "fmt"
    "log"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(databaseURL, migrationsPath string) error {
    m, err := migrate.New(
        fmt.Sprintf("file://%s", migrationsPath),
        databaseURL,
    )
    if err != nil {
        return fmt.Errorf("failed to create migrate instance: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("failed to run migrations: %w", err)
    }

    log.Println("Migrations applied successfully")
    return nil
}
```

Use it in your main function:

```go
func main() {
    dbURL := "postgres://myapp_user:password@localhost:5432/myapp?sslmode=disable"

    // Run migrations on startup
    if err := database.RunMigrations(dbURL, "./migrations"); err != nil {
        log.Fatalf("Migration failed: %v", err)
    }

    // Continue with normal application startup
    // ...
}
```

## Migration Best Practices

**1. Never modify an existing migration**

Once a migration has been run (by you or anyone else), do not change it. If you need to fix something, create a new migration. Modifying an existing migration will cause the version to be out of sync.

**2. Always write a down migration**

Every up migration should have a corresponding down migration. Test the down migration to make sure it works. If you cannot undo a change, you cannot roll back safely.

**3. Keep migrations small**

One migration per logical change. Do not put 10 table creations in one migration file. If something goes wrong, you want to know exactly which change caused the problem.

**4. Test both directions**

Always test that running up and then down returns the database to its previous state:

```bash
migrate up    # apply
migrate down 1 # undo
migrate up    # re-apply
```

**5. Be careful with data migrations**

Schema changes (CREATE, ALTER, DROP) are safe. Data migrations (INSERT, UPDATE, DELETE) are risky. If you need to update existing data, write a separate script and test it carefully.

**6. Use transactions when possible**

PostgreSQL runs each migration in a transaction by default. This means if a migration fails halfway, all changes are rolled back. But some operations cannot run inside a transaction (like CREATE INDEX CONCURRENTLY). Be aware of this.

## Adding a Column Migration

Here is a real example of adding a column to an existing table:

```sql
-- 000004_add_image_url_to_products.up.sql
ALTER TABLE products ADD COLUMN image_url TEXT;
```

```sql
-- 000004_add_image_url_to_products.down.sql
ALTER TABLE products DROP COLUMN image_url;
```

Simple, clean, and reversible. Run `migrate up` and the column appears. Run `migrate down 1` and it disappears.

## Dealing with Dirty Migrations

Sometimes a migration fails halfway. The database is in an inconsistent state. golang-migrate marks the migration as **dirty**. You will see an error like:

```
Dirty database
```

To fix this:

1. Check what state the database is actually in
2. Either manually complete the migration or manually undo it
3. Force the migration version to the correct number:

```bash
migrate force 3
```

This tells golang-migrate "trust me, version 3 is the current state." Use this carefully.

## My Experience with Migrations

I used to manage my database schema manually. ALTER TABLE commands in psql, no records of what I did. It was fine until I started working with a team. One person added a column, another person did not know, code broke everywhere.

Migrations fixed this completely. Every schema change is tracked. Anyone can run `migrate up` and their database matches the code. Rolling back is one command. It is one of those things that seems like extra work until you try it, and then you cannot imagine working without it.
