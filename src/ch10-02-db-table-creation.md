# Database & Table Creation

## Creating Tables in PostgreSQL

Tables are where your data lives. Before you can insert or query anything, you need to create tables with the right structure. A well-designed table makes your application easier to build and maintain.

## Common PostgreSQL Data Types

Here are the data types you will use most often:

| PostgreSQL Type | Go Type | Description |
|-----------------|---------|-------------|
| **VARCHAR(n)** | string | Variable-length text with a max length |
| **TEXT** | string | Unlimited text |
| **INTEGER** | int | Whole number |
| **BIGINT** | int64 | Large whole number |
| **SERIAL** | int | Auto-incrementing integer |
| **BIGSERIAL** | int64 | Auto-incrementing big integer |
| **BOOLEAN** | bool | True or false |
| **TIMESTAMP** | time.Time | Date and time |
| **DATE** | time.Time | Date only |
| **UUID** | [16]byte | Universally unique identifier |
| **NUMERIC** | float64 | Exact decimal number |
| **JSON/JSONB** | map/struct | JSON data |

**VARCHAR vs TEXT**: In PostgreSQL, there is no performance difference. Use VARCHAR when you want to enforce a max length. Use TEXT for unlimited length.

**SERIAL vs BIGSERIAL**: Use SERIAL for most tables. Use BIGSERIAL if you expect more than 2 billion rows.

**JSON vs JSONB**: JSONB is binary JSON. It is slower to insert but faster to query. Almost always prefer JSONB over JSON.

## Primary Keys

A **primary key** uniquely identifies each row in a table. Every table should have one:

```sql
-- Using SERIAL (auto-incrementing integer)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- Using UUID (universally unique identifier)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL
);
```

I prefer **SERIAL** for most tables because it is simple and efficient. UUIDs are useful when you need to generate IDs on the client side or across distributed systems.

## Foreign Keys

A **foreign key** creates a relationship between two tables. It ensures that a value in one table exists in another table:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    total NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The `REFERENCES users(id)` part means that every `user_id` in the orders table must exist as an `id` in the users table. PostgreSQL enforces this. You cannot insert an order for a user that does not exist.

You can also specify what happens when the referenced row is deleted:

```sql
-- Delete orders when user is deleted
user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE

-- Set user_id to NULL when user is deleted
user_id INTEGER REFERENCES users(id) ON DELETE SET NULL

-- Prevent deleting a user who has orders (default)
user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE RESTRICT
```

## Constraints

**Constraints** enforce rules on your data. They keep your database clean and consistent:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,                    -- cannot be empty
    price NUMERIC(10, 2) NOT NULL CHECK (price > 0), -- must be positive
    sku VARCHAR(50) UNIQUE,                         -- no duplicates
    stock INTEGER NOT NULL DEFAULT 0,               -- default value
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP  -- auto-set timestamp
);
```

**NOT NULL** - The column must have a value. Missing values are not allowed.

**UNIQUE** - No two rows can have the same value in this column.

**CHECK** - The value must satisfy a condition. Great for business rules.

**DEFAULT** - A value used when none is provided.

## Creating Ecommerce Tables

Let us create a complete set of tables for an ecommerce application:

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
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

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order items table (many-to-many relationship)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price NUMERIC(10, 2) NOT NULL,  -- price at time of purchase
    CONSTRAINT unique_order_product UNIQUE (order_id, product_id)
);
```

Notice the design decisions:

- **order_items** is a separate table that connects orders and products. This is how you handle many-to-many relationships.
- **price** in order_items is copied from products at purchase time. If the product price changes later, the order still shows the correct historical price.
- **ON DELETE CASCADE** on order_id means deleting an order also deletes its items.
- **ON DELETE RESTRICT** on product_id means you cannot delete a product that has been ordered.
- The **unique_order_product** constraint prevents the same product from appearing twice in one order.

## Adding Indexes

Indexes make queries faster. Add them on columns you frequently search or filter by:

```sql
-- Index on email for fast login lookups
CREATE INDEX idx_users_email ON users(email);

-- Index on user_id for fast order lookups
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Index on order_id for fast order item lookups
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Index on category for product filtering
CREATE INDEX idx_products_category ON products(category);
```

Indexes trade insert/update speed for query speed. Do not add indexes on every column. Add them on columns that appear in WHERE clauses and JOIN conditions.

## Modifying Tables

Sometimes you need to change a table after creating it:

```sql
-- Add a column
ALTER TABLE products ADD COLUMN image_url TEXT;

-- Remove a column
ALTER TABLE products DROP COLUMN image_url;

-- Change a column type
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12, 2);

-- Add a constraint
ALTER TABLE products ADD CONSTRAINT positive_stock CHECK (stock >= 0);

-- Rename a column
ALTER TABLE products RENAME COLUMN name TO product_name;
```

For production databases, use migrations instead of direct ALTER statements. We will cover migrations later in this section.
