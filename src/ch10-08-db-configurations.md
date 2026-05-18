# Database Configurations

## Why Configuration Matters

In development, you might hardcode your database connection string. That is fine for local testing. But in production, things are different. You need different credentials, different SSL settings, different pool sizes. Hardcoding any of this is a security risk and a maintenance nightmare.

Good configuration separates **what changes** from **what stays the same**. Your Go code stays the same. The configuration changes per environment.

## Environment Variables

The most common way to configure a database is with **environment variables**:

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
    ConnMaxIdleTime time.Duration
}

func LoadConfig() Config {
    return Config{
        Host:            getEnv("DB_HOST", "localhost"),
        Port:            getEnv("DB_PORT", "5432"),
        User:            getEnv("DB_USER", "postgres"),
        Password:        getEnv("DB_PASSWORD", ""),
        Name:            getEnv("DB_NAME", "myapp"),
        SSLMode:         getEnv("DB_SSLMODE", "disable"),
        MaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
        MaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 25),
        ConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
        ConnMaxIdleTime: getEnvDuration("DB_CONN_MAX_IDLE_TIME", 10*time.Minute),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    value := os.Getenv(key)
    if value == "" {
        return defaultValue
    }
    var n int
    fmt.Sscanf(value, "%d", &n)
    return n
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
    value := os.Getenv(key)
    if value == "" {
        return defaultValue
    }
    d, err := time.ParseDuration(value)
    if err != nil {
        return defaultValue
    }
    return d
}
```

## Using .env Files

Setting environment variables in the terminal is tedious. The **godotenv** package lets you load them from a `.env` file:

```bash
go get github.com/joho/godotenv
```

Create a `.env` file in your project root:

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=myapp_user
DB_PASSWORD=secure_password
DB_NAME=myapp
DB_SSLMODE=disable
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=25
DB_CONN_MAX_LIFETIME=5m
DB_CONN_MAX_IDLE_TIME=10m
```

Then load it in your application:

```go
package main

import (
    "log"
    "github.com/joho/godotenv"
)

func main() {
    // Load .env file (ignore error in production where env vars are set externally)
    err := godotenv.Load()
    if err != nil {
        log.Println("No .env file found, using system environment variables")
    }

    // Now load the config
    cfg := database.LoadConfig()
    // ...
}
```

**Important**: Never commit `.env` files to version control. Add `.env` to your `.gitignore`. Include a `.env.example` file instead:

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=your_database
DB_SSLMODE=disable
```

## Connection String Format

PostgreSQL connection strings come in two formats:

**URL format** (recommended):
```
postgres://user:password@host:port/database?sslmode=disable
```

**Key-value format**:
```
host=localhost port=5432 user=myapp_user password=secure_password dbname=myapp sslmode=disable
```

I prefer the URL format because it is more readable and consistent with other services.

## SSL Mode

SSL mode controls whether the connection is encrypted:

| SSL Mode | Description |
|----------|-------------|
| **disable** | No encryption. Only for local development. |
| **allow** | Prefer non-SSL, but allow SSL. |
| **prefer** | Prefer SSL, but allow non-SSL. |
| **require** | Require SSL. Reject if SSL is not available. |
| **verify-ca** | Require SSL and verify the server certificate. |
| **verify-full** | Require SSL and verify the certificate and hostname. |

For development, use **disable**. For production, use at least **require**, preferably **verify-full**.

## Connection Pool Configuration

The connection pool is critical for production performance. Here is a detailed guide:

```go
func Connect(cfg Config) (*sql.DB, error) {
    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=%s",
        cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.Name, cfg.SSLMode,
    )

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    // Pool configuration
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    db.SetConnMaxIdleTime(cfg.ConnMaxIdleTime)

    // Verify the connection
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return db, nil
}
```

How to choose pool sizes:

- **MaxOpenConns**: Start with 25. Set based on your database max connections divided by number of app instances.
- **MaxIdleConns**: Same as MaxOpenConns to avoid connection churn.
- **ConnMaxLifetime**: 5 minutes is a good default. Set shorter if your database has connection timeouts.
- **ConnMaxIdleTime**: 10 minutes is fine. Frees connections when traffic drops.

## Logging Queries

For debugging, you might want to log every query. You can wrap the database with a logging middleware:

```go
type LoggingDB struct {
    db *sql.DB
}

func (l *LoggingDB) Exec(query string, args ...interface{}) (sql.Result, error) {
    start := time.Now()
    result, err := l.db.Exec(query, args...)
    duration := time.Since(start)
    log.Printf("EXEC %s | args=%v | duration=%v | err=%v", query, args, duration, err)
    return result, err
}
```

For production, use a proper query logger or database driver that supports logging.

## Health Check Endpoint

Every production application should have a health check endpoint that verifies the database connection:

```go
func HealthCheck(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()

        if err := db.PingContext(ctx); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{
                "status":   "unhealthy",
                "database": "disconnected",
                "error":    err.Error(),
            })
            return
        }

        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status":   "healthy",
            "database": "connected",
        })
    }
}
```

## Complete Configuration Example

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"
    "time"

    "github.com/joho/godotenv"
    _ "github.com/lib/pq"
)

func main() {
    // Load .env
    godotenv.Load()

    // Load config from environment
    cfg := LoadConfig()

    // Connect to database
    db, err := Connect(cfg)
    if err != nil {
        log.Fatalf("Database connection failed: %v", err)
    }
    defer db.Close()

    log.Println("Connected to PostgreSQL successfully")
    log.Printf("Pool: MaxOpen=%d, MaxIdle=%d, Lifetime=%v",
        cfg.MaxOpenConns, cfg.MaxIdleConns, cfg.ConnMaxLifetime)
}
```

Good configuration makes your application portable. The same binary runs in development, staging, and production. Only the environment variables change.
