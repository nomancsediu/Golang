# Chapter 4: Data Structures & Core Go

## What This Section Is About

This section is where things start to feel real. We are going to cover the data structures and core language features that you will use every single day when writing Go. These are not theoretical concepts. These are the tools that show up in every Go file you will ever write.

Here is what we will go through:

- **Structs** - Go's way of grouping related data together. No classes, no inheritance. Just simple, powerful structs.
- **Receiver Methods** - How you attach behavior to your types. This is Go's answer to object-oriented programming, but simpler.
- **Arrays** - Fixed-size collections. You need to understand them, but you will rarely use them directly.
- **Pointers** - Memory addresses. Go keeps them simple and safe compared to C.
- **Slices** - The dynamic, flexible cousin of arrays. This is the data structure you will use the most in Go.
- **Bogus Data Types** - Type definitions and aliases that help you write safer, more expressive code.
- **Defer** - Go's built-in mechanism for cleanup. It makes resource management reliable and predictable.

## Why These Topics Matter

When I first started learning Go, I jumped straight into building HTTP servers and writing API handlers. That was fun, but I kept running into confusion. Why does my struct not update when I call a method? Why does my slice sometimes share data with another slice? What does `&` actually do?

The answer to all of those questions lives right here in this section. These are the building blocks. Everything else in Go, from HTTP handlers to database operations to concurrency patterns, is built on top of these fundamentals.

If you skip this section, you will write Go code that compiles but behaves in surprising ways. You will spend hours debugging issues that trace back to not understanding value vs pointer semantics, or slice internals, or how defer works. I know because that is exactly what happened to me.

## How These Concepts Connect

These topics are not isolated. They form a connected web of ideas that make Go work the way it does:

**Structs** hold your data. **Methods** give that data behavior. But to decide whether a method should use a value or pointer receiver, you need to understand **pointers**. And when you work with collections of structs, you will use **slices**. Slices are built on top of **arrays**, and they use pointers internally. To keep your resources clean, you use **defer**. And to make your types safer and more expressive, you use **bogus data types**.

Here is a quick preview of how these pieces fit together:

```go
// A type-safe ID using a bogus data type
type UserID string

// A struct to hold user data
type User struct {
    ID    UserID
    Name  string
    Email string
}

// A method with a pointer receiver to modify the user
func (u *User) UpdateEmail(email string) {
    u.Email = email
}

// A function that uses slices and defer together
func FindUsersByEmail(domain string) ([]User, error) {
    db, err := OpenDatabase()
    if err != nil {
        return nil, err
    }
    defer db.Close() // reliable cleanup with defer

    // users is a slice of User structs
    users := []User{}
    // ... query the database ...
    return users, nil
}
```

See how everything connects? Bogus types, structs, pointer receivers, slices, and defer all in one short example. This is typical Go code.

## Go's Data Structures: Simple but Powerful

One thing I really like about Go is how **simple** its data structures are compared to other languages. There are no classes, no constructors, no inheritance chains, no generics with wild constraints (well, generics arrived in Go 1.18, but they are still simple). You have structs to hold data, methods to add behavior, and slices for dynamic collections. That covers 90% of what you need.

In Python, you might use a class with `__init__`, properties, decorators, and metaclasses. In Java, you have classes, interfaces, abstract classes, and design patterns everywhere. In Go, you just define a struct and attach methods to it. It feels refreshingly straightforward.

But do not confuse simple with weak. Go's data structures are incredibly powerful when you understand how they work under the hood. A slice is not just a list. It is a view into an array with length and capacity. A struct is not just a record. With embedding and methods, it can express complex relationships. A pointer is not just a memory address. It is the key to understanding when data is shared versus copied.

## The Value vs Reference Distinction

One theme that runs through this entire section is the difference between **value types** and **reference types** in Go. This is something that confused me a lot at first, so let me give you a heads up:

- **Arrays and structs are value types.** When you assign them or pass them to a function, Go makes a copy. Changes to the copy do not affect the original.
- **Slices, maps, and channels are reference types.** When you assign them or pass them to a function, you are sharing the same underlying data. Changes affect everyone who holds a reference.
- **Pointers** let you turn value semantics into reference semantics. Instead of copying the data, you pass its address.

This distinction explains so many "why did my data change?" bugs. Once you understand it, a lot of Go's behavior starts making sense.

## A Personal Note

I used to think that "simple" meant "limited." Go changed my mind. The simplicity here is intentional. It means there are fewer ways to shoot yourself in the foot. It means you can read someone else's Go code and actually understand what it does. It means the language works with you, not against you.

That said, there are some sharp edges. Slices and pointers in particular can trip you up if you do not understand how they work internally. This section will help you avoid those pitfalls.

My advice as you go through these chapters: **type out every code example yourself.** Do not just read them. Run them. Modify them. Break them. See what happens. That is how these concepts will move from "I kind of understand" to "I actually get it."

Let us get started with **structs**, the foundation of everything in Go.
