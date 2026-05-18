# Ch03 02 Named Functions

Named functions are the functions you have been writing since the beginning. A **named function** is simply a function that has a name. You declare it with the `func` keyword followed by the name, parameters, and body. Seems simple, right? But there is more to naming functions than you might think.

## Declaring a Named Function

Here is the basic syntax you already know:

```go
func greet(name string) string {
    return "Hello, " + name
}
```

The function name is `greet`. You can call it from anywhere in the same package. That is the simplest form.

## Why Naming Matters

A good function name tells you **what** the function does without needing to read the body. A bad function name forces you to read the implementation to understand it. This matters a lot when you are reading code you wrote three months ago, or code written by someone else.

Compare these two:

```go
// Bad: what does "process" mean? Process what?
func process(data string) error {
    return validate(data)
}

// Good: the name tells you exactly what it does
func validateUserInput(data string) error {
    return validate(data)
}
```

The second name is much clearer. When you see `validateUserInput` called somewhere, you know what it does immediately.

## Exported vs Unexported Functions

This is a big deal in Go. Go uses a simple rule to control visibility:

- **Exported** functions start with an **uppercase letter**. They can be called from other packages.
- **Unexported** functions start with a **lowercase letter**. They can only be called within the same package.

```go
package user

// Exported: other packages can call this
func CreateUser(name string) *User {
    return &User{Name: name}
}

// Unexported: only this package can call this
func hashPassword(pwd string) string {
    // internal logic
    return "hashed_" + pwd
}
```

If you try to call `hashPassword` from another package, you will get a compile error. Go enforces this strictly. I like this because it makes the API of a package very clear. If a function starts with an uppercase letter, it is public. If it starts with lowercase, it is private. No extra keywords needed.

## Functions as Package API

When you write a package, the **exported functions** form its API. Everything else is implementation detail. Think of it this way: exported functions are promises you make to the users of your package. Unexported functions are your internal helpers that you can change anytime without breaking anyone.

```go
package mathutils

// Add is an exported function. This is part of the package API.
// Other packages rely on this, so think carefully before changing it.
func Add(a, b int) int {
    return a + b
}

// computeChecksum is unexported. Only this package uses it.
// You can change the implementation without breaking anyone.
func computeChecksum(data []byte) uint32 {
    // internal implementation
    return 0
}
```

A good package has a small, clear set of exported functions. Too many exported functions means the package is hard to understand. Too few means it is not useful.

## Method Names on Structs

When you attach a function to a struct (called a **method** in Go), the naming matters even more. Methods are how users interact with your types. Good method names read like natural language.

```go
type User struct {
    Name string
    Age  int
}

// Good: reads naturally as "user.FullName()"
func (u User) FullName() string {
    return u.Name
}

// Good: reads naturally as "user.IsAdult()"
func (u User) IsAdult() bool {
    return u.Age >= 18
}
```

Some naming conventions for methods in Go:

- **Getter methods** do not use the `Get` prefix. Use `Name()` not `GetName()`.
- **Boolean methods** start with `Is`, `Has`, or `Can`. Like `IsAdult()`, `HasPermission()`.
- **Mutating methods** use strong verbs. Like `Add()`, `Remove()`, `Update()`.

## Good vs Bad Naming Examples

Here are some more examples to show the difference:

```go
// BAD NAMES
func calc(x, y int) int { ... }      // calc what?
func doStuff() { ... }               // do what stuff?
func getData() []byte { ... }        // what data?
func handleIt(err error) { ... }     // handle what exactly?

// GOOD NAMES
func calculateTotal(price, tax int) int { ... }
func sendWelcomeEmail(user string) { ... }
func fetchUserProfile(id int) []byte { ... }
func handleDatabaseError(err error) { ... }
```

The good names are longer, but they save everyone time. You read the name and you know what the function does. No guessing.

## Personal Note

I used to write short function names because I thought they were cleaner. Then I worked on a project with hundreds of functions and realized I could not remember what half of them did. Long, descriptive names are not verbose. They are documentation. In Go, the convention is clear: be descriptive. A function called `Process` is almost always a bad name. A function called `ProcessPaymentRequest` is a good name. Your future self will thank you.
