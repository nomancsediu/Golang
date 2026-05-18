# Chapter 11: Architecture & Clean Code

## Now That You Can Build Things, Let Us Build Them Well

You have learned the building blocks. You can write Go code. You can connect to a database. You can build APIs with authentication. You can use concurrency. You understand interfaces and design patterns.

Now comes the hard part: putting it all together in a way that makes sense.

Architecture is what separates a side project from a production system. A side project works when everything goes right. A production system works when things go wrong. When the database is slow. When a third-party API goes down. When you need to add a new feature without breaking the old ones.

## What This Section Covers

This section is about the practical patterns that make real Go applications maintainable:

1. **DDD Into Code** - Taking the domain driven design concepts from the previous section and implementing them in a real project structure with proper layers and dependency direction.

2. **Multilingual System Design** - How to build APIs that support multiple languages. More applications need this than you might think, especially if you are building for a global audience.

3. **Pagination** - Returning 1 million rows in one API call is a bad idea. We will learn how to paginate properly with limit and offset.

4. **Query Parameters** - Building flexible APIs that accept filters, sorting, and search from query parameters.

5. **Offset Pagination** - The most common pagination approach, how it works under the hood, and when it is the right choice versus cursor-based pagination.

6. **Map Usage** - Go's map type is incredibly useful. We will cover common patterns for using maps effectively, including concurrent access.

## My Personal Reflection

I used to think architecture was something only "senior developers" worried about. I just wanted to write code that works. But as my projects grew, I kept running into the same problems. Code was hard to change. Features took longer and longer to add. Tests were impossible to write. I was spending more time fighting my own code than building new features.

The patterns in this section are not theoretical. They are practical solutions to problems I have actually faced. Pagination was the first thing I had to add to every API. Query parameters were the second. Multilingual support surprised me when my first "real" project needed it. Map patterns I use every single day.

## What Good Architecture Looks Like

Good architecture has a few key properties:

**Separation of concerns** - Each piece of code does one thing well. Handlers handle HTTP. Services handle business logic. Repositories handle data storage. When you need to change how an order is processed, you go to the service. When you need to change how data is stored, you go to the repository. You never have to search through 50 files.

**Dependency direction** - Dependencies point inward. The domain does not depend on the database. The database depends on the domain. This means you can change your database without touching your business logic. You can change your web framework without touching your services.

**Testability** - You can test any layer in isolation. No test requires a real database, a real email server, or a real payment gateway. Every dependency can be mocked. Every test is fast and deterministic.

**Changeability** - When a requirement changes, you change one part of the code. The change does not ripple across the entire codebase. New features fit into the existing structure naturally.

## A Simple Example

Here is what bad architecture looks like:

```go
// Everything in one handler. Hard to test. Hard to change.
func CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    json.NewDecoder(r.Body).Decode(&req)

    db, _ := sql.Open("postgres", "postgres://localhost/mydb")
    defer db.Close()

    db.Exec("INSERT INTO orders ...")
    smtp.SendMail("smtp.gmail.com", nil, "from@shop.com", []string{req.Email}, []byte("Order confirmed"))
    http.Post("https://payments.example.com/charge", "application/json", nil)

    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

And here is what good architecture looks like:

```go
// Each concern is in its own layer. Easy to test. Easy to change.
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    json.NewDecoder(r.Body).Decode(&req)

    order, err := h.service.CreateOrder(req)
    if err != nil {
        respondError(w, err)
        return
    }

    respondJSON(w, http.StatusCreated, order)
}
```

The handler only handles HTTP. The service handles business logic. The repository handles storage. The email sender handles notifications. Each piece is simple, testable, and replaceable.

Architecture is not about making things complex. It is about making things simple enough that you can still understand them six months from now. It is about making your code easy to change, easy to test, and easy to extend.

The best architecture is one where adding a new feature feels natural, not like hacking your way through a jungle of tangled code.

Let us dive in and learn how to build Go applications that last.
