# Database Connection Setup in Go

## The database/sql Package

Go's standard library includes the **database/sql** package. It provides a generic interface around SQL databases. You write the same code regardless of which database you are using. The only thing that changes is the driver.

This is powerful. You can switch from PostgreSQL to MySQL by changing one import and the connection string. All your query code stays the same.

## Installing a PostgreSQL Driver

Go needs a driver to talk to PostgreSQL. There are two popular options:

**lib/pq** - The original PostgreSQL driver. Stable and well-tested.
```bash
go get github.com/lib/pq
```

**pgx** - A newer driver with better performance and more features.
```bash
go get github.com/jackc/pgx/v5/stdlib
```

I will use **lib/pq** in these examples because it is simpler to get started with. For production, consider pgx for better performance.

## Opening a Connection

Here is how you open a database connection:

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq" // driver import (underscore means we use it indirectly)
)

func main() {
    // Connection string
    connStr := "postgres://myapp_user:secure_password@localhost:5432/myapp?sslmode=disable"

    // Open the connection
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatalf("Failed to open database: %v", err)
    }
    // Always close the database when done
    defer db.Close()

    // Test the connection
    err = db.Ping()
    if err != nil {
        log.Fatalf("Failed to ping database: %v", err)
    }

    fmt.Println("Connected to PostgreSQL!")
}
```

Important: **sql.Open does not actually connect to the database**. It just validates the arguments. The real connection happens lazily when you first execute a query, or when you call **db.Ping()**. Always call Ping to verify the connection works.

## Connection String Format

The connection string format is:

```
postgres://username:password@host:port/database?sslmode=disable
```

Parameters:
- **username** - Your PostgreSQL user
- **password** - Your PostgreSQL password
- **host** - Usually localhost for development
- **port** - Default is 5432
- **database** - The database name
- **sslmode** - disable for local development, require for production

## Using Environment Variables

Never hardcode database credentials in your code. Use environment variables:

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/lib/pq"
)

func connectDB() (*sql.DB, error) {
    // Read from environment variables with defaults
    host := getEnv("DB_HOST", "localhost")
    port := getEnv("DB_PORT", "5432")
    user := getEnv("DB_USER", "myapp_user")
    password := getEnv("DB_PASSWORD", "secure_password")
    dbname := getEnv("DB_NAME", "myapp")
    sslmode := getEnv("DB_SSLMODE", "disable")

    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=%s",
        user, password, host, port, dbname, sslmode,
    )

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return db, nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

## Connection Pool Configuration

The `database/sql` package manages a **connection pool** automatically. But you should configure it for production:

```go
func setupDB() (*sql.DB, error) {
    db, err := connectDB()
    if err != nil {
        return nil, err
    }

    // Maximum number of open connections to the database
    // Default is 0 (unlimited). Set this to prevent overwhelming the database.
    db.SetMaxOpenConns(25)

    // Maximum number of idle connections in the pool
    // Keep some connections ready to avoid connection setup overhead
    db.SetMaxIdleConns(25)

    // Maximum time a connection can be reused
    // Connections get stale over time. Replace them periodically.
    db.SetConnMaxLifetime(5 * time.Minute)

    // Maximum time a connection can be idle
    // Close connections that have been idle too long
    db.SetConnMaxIdleTime(10 * time.Minute)

    return db, nil
}
```

Why these values?

- **MaxOpenConns(25)** - Limits how many connections your app can make. Too many connections slow down the database. Start with 25 and adjust based on your load.
- **MaxIdleConns(25)** - Same as MaxOpenConns. This avoids creating and destroying connections constantly.
- **ConnMaxLifetime(5 min)** - Prevents stale connections. Also helps if the database server closes idle connections.
- **ConnMaxIdleTime(10 min)** - Frees up resources when traffic is low.

## Closing the Connection

Always close the database connection when your program exits:

```go
func main() {
    db, err := setupDB()
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close() // This runs when main() returns

    // Your application logic here
}
```

The `defer db.Close()` ensures the connection is closed even if your program panics. In a web server, you typically close the database in the shutdown handler.

## Complete Connection Package

Here is a reusable connection package I use in my projects:

```go
package database

import (
    "database/sql"
    "fmt"
    "os"
    "time"

    _ "github.com/lib/pq"
)

type Config struct {
    Host            string
    Port            string
    User            string
    Password        string
    Name            string
    SSLMode         string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

func DefaultConfig() Config {
    return Config{
        Host:            getEnv("DB_HOST", "localhost"),
        Port:            getEnv("DB_PORT", "5432"),
        User:            getEnv("DB_USER", "myapp_user"),
        Password:        getEnv("DB_PASSWORD", "secure_password"),
        Name:            getEnv("DB_NAME", "myapp"),
        SSLMode:         getEnv("DB_SSLMODE", "disable"),
        MaxOpenConns:    25,
        MaxIdleConns:    25,
        ConnMaxLifetime: 5 * time.Minute,
    }
}

func Connect(cfg Config) (*sql.DB, error) {
    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=%s",
        cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.Name, cfg.SSLMode,
    )

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return db, nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

This makes it easy to connect to any PostgreSQL database with sensible defaults.
