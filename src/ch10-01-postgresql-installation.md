# PostgreSQL Installation

## Installing PostgreSQL

Before we can write any Go code, we need PostgreSQL installed and running. Here is how to install it on different operating systems.

### macOS

The easiest way on macOS is with **Homebrew**:

```bash
# Install PostgreSQL
brew install postgresql@16

# Start the service
brew services start postgresql@16

# Verify it is running
psql --version
```

### Ubuntu/Debian

```bash
# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib

# Start the service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify it is running
psql --version
```

### Windows

Download the installer from the official PostgreSQL website: https://www.postgresql.org/download/windows/

Run the installer and follow the setup wizard. Make sure to remember the password you set for the postgres user.

### Docker (Any OS)

If you prefer Docker, this is the cleanest option:

```bash
# Run PostgreSQL in a container
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -d postgres:16

# Verify it is running
docker ps
```

I use Docker for development. It keeps my machine clean and makes it easy to reset the database.

## Basic Setup

After installing PostgreSQL, you need to create a user and a database. By default, PostgreSQL creates a user called `postgres`.

### Using psql Command Line

**psql** is the interactive terminal for PostgreSQL. You will use it a lot.

```bash
# Connect to PostgreSQL as the postgres user
sudo -u postgres psql

# Or if you set a password during install
psql -U postgres -h localhost
```

### Create a New User

```sql
-- Create a new user for your application
CREATE USER myapp_user WITH PASSWORD 'secure_password';

-- Give the user permission to create databases
ALTER USER myapp_user CREATEDB;
```

### Create a New Database

```sql
-- Create a database for your application
CREATE DATABASE myapp OWNER myapp_user;

-- Connect to the new database
\c myapp
```

## Common psql Commands

These are the psql commands I use every day:

```sql
-- List all databases
\l

-- Connect to a database
\c myapp

-- List all tables in the current database
\dt

-- List all users
\du

-- Describe a table structure
\d users

-- Show query history
\s

-- Execute a SQL file
\i /path/to/file.sql

-- Quit psql
\q
```

## Connecting with a GUI Tool

If you prefer a graphical interface over the command line, here are some options:

**pgAdmin** - The official PostgreSQL GUI. It is powerful but can be slow. Install it from https://www.pgadmin.org/

**DBeaver** - A universal database tool that works with many databases. My personal favorite. Install it from https://dbeaver.io/

**DataGrip** - A paid IDE from JetBrains. Great if you already use JetBrains products.

To connect with any of these tools, you need:

- **Host**: localhost
- **Port**: 5432
- **Database**: myapp
- **Username**: myapp_user
- **Password**: secure_password

## Basic SQL Commands

Here are the SQL commands you will use most often:

```sql
-- Create a table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert a row
INSERT INTO users (email, name) VALUES ('john@example.com', 'John');

-- Query all rows
SELECT * FROM users;

-- Query with a condition
SELECT * FROM users WHERE email = 'john@example.com';

-- Update a row
UPDATE users SET name = 'Johnny' WHERE id = 1;

-- Delete a row
DELETE FROM users WHERE id = 1;

-- Count rows
SELECT COUNT(*) FROM users;
```

## Verifying Your Setup

After installation, run this quick test to make sure everything works:

```bash
# Connect to your database
psql -U myapp_user -d myapp -h localhost

# Create a test table
CREATE TABLE test (id SERIAL PRIMARY KEY, message TEXT);

# Insert test data
INSERT INTO test (message) VALUES ('Hello, PostgreSQL!');

# Query it back
SELECT * FROM test;

-- Expected output:
-- id | message
-- ---+-------------------
--  1 | Hello, PostgreSQL!

-- Clean up
DROP TABLE test;

-- Exit
\q
```

If you see "Hello, PostgreSQL!" in the output, your PostgreSQL installation is working correctly. Now we are ready to connect to it from Go.
