# Chapter 10: Database with Go

## Real Applications Need Real Data

Every real application needs to store data. Users need accounts. Products need catalogs. Orders need records. You can only get so far with in-memory data and flat files. Eventually, you need a real database.

Learning database operations in Go was one of the most practical things I have done. It is not glamorous. It is not flashy. But it is what makes your application actually useful. Without a database, your app forgets everything when you restart it.

## Why PostgreSQL?

I chose **PostgreSQL** for this section because it is the most popular open-source relational database. It is powerful, reliable, and widely used in production. It has excellent support for JSON, full-text search, and complex queries. And it works beautifully with Go.

There are other options: MySQL, SQLite, MongoDB. But PostgreSQL is the one I see most often in Go backend jobs. If you know PostgreSQL, you can work with almost any Go backend team.

## What This Section Covers

This section walks through everything you need to work with databases in Go:

1. **PostgreSQL Installation** - Getting PostgreSQL installed and set up on your machine.

2. **Database & Table Creation** - Creating databases and tables with proper data types, primary keys, and foreign keys.

3. **Database Connection Setup** - Connecting to PostgreSQL from Go using `database/sql` and a driver.

4. **Create Operation** - Inserting data into the database using `db.Exec()` and parameterized queries.

5. **Find Operation** - Querying data from the database using `db.Query()` and `db.QueryRow()`.

6. **Database Data Types** - Mapping PostgreSQL types to Go types and handling special types like JSON, UUID, and arrays.

7. **CRUD Operations** - Complete Create, Read, Update, Delete with the repository pattern.

8. **Database Configurations** - Environment variables, connection pooling, and production settings.

9. **Database Migrations** - Version controlling your database schema with golang-migrate.

## My Personal Experience

I remember my first time connecting Go to a database. It was confusing. There were multiple drivers to choose from. The `database/sql` API felt different from what I was used to in other languages. Handling NULL values was tricky. Getting the connection string right took forever.

But once I understood the basics, everything clicked. The `database/sql` package is actually very well designed. It handles connection pooling automatically. It prevents SQL injection with parameterized queries. It works the same way regardless of which database you use.

The most important thing I learned: **always use parameterized queries**. Never concatenate user input into SQL strings. This is not just about security. It is about writing correct, reliable code.

## The Repository Pattern

Throughout this section, you will see the **repository pattern** a lot. A repository is an interface that hides the database details from your business logic. Instead of writing SQL queries in your handlers, you call methods like `repo.FindByID()` and `repo.Save()`.

This pattern has three big benefits:

- **Testability** - You can mock the repository in tests without a real database
- **Flexibility** - You can swap PostgreSQL for another database by implementing the same interface
- **Clarity** - Your handlers focus on HTTP concerns, not SQL queries

Here is what the pattern looks like in code:

```go
// The interface your business logic depends on
type ProductRepository interface {
    FindByID(id int64) (*Product, error)
    Save(product *Product) error
    FindAll() ([]*Product, error)
}

// A concrete implementation with PostgreSQL
type PostgresProductRepo struct {
    db *sql.DB
}

func (r *PostgresProductRepo) FindByID(id int64) (*Product, error) {
    // SQL query here
}

func (r *PostgresProductRepo) Save(product *Product) error {
    // SQL query here
}
```

We will build up to this pattern gradually. First, we will learn the raw database operations. Then we will wrap them in a repository interface. This way, you understand what the repository is doing under the hood.

## A Note on SQL

This section assumes you know basic SQL. If you do not, here is a quick primer:

- **SELECT** reads data from a table
- **INSERT** adds new data to a table
- **UPDATE** modifies existing data
- **DELETE** removes data
- **CREATE TABLE** creates a new table
- **WHERE** filters results
- **JOIN** combines data from multiple tables

You do not need to be a SQL expert. Basic SELECT, INSERT, UPDATE, and DELETE will get you through this section. The more complex queries, we will build together.

Let us start from the beginning and build up our database skills step by step.
