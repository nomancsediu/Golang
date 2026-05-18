# Functions Introduction in Go

## What Are Functions?

A **function** is a reusable block of code that does one specific task. Instead of writing the same code over and over, you put it in a function and call it whenever you need it. Functions are the building blocks of every Go program. Even the simplest program we wrote had one: `func main()`.

Let me share what I have learned about how functions work in Go.

## Defining a Function

In Go, you define a function using the `func` keyword:

```go
func greet() {
    fmt.Println("Hello there!")
}
```

This is the simplest form: a function with no parameters and no return value. You call it by writing its name followed by parentheses:

```go
greet() // prints: Hello there!
```

## Functions with Parameters

Most functions need some input to work with. These inputs are called **parameters**:

```go
func greetPerson(name string) {
    fmt.Println("Hello,", name)
}
```

When you call this function, you pass an **argument**:

```go
greetPerson("Noman")  // prints: Hello, Noman
greetPerson("Alice")  // prints: Hello, Alice
```

### Multiple Parameters

A function can have multiple parameters. Each parameter must have a type:

```go
func add(a int, b int) {
    fmt.Println("Sum:", a+b)
}
```

If multiple parameters share the same type, you can write the type only once at the end:

```go
func add(a, b int) {
    fmt.Println("Sum:", a+b)
}
```

Both `a` and `b` are `int`. This shorthand is common in Go and makes the code cleaner.

### Parameters with Different Types

When parameters have different types, you must specify each type:

```go
func printInfo(name string, age int) {
    fmt.Printf("%s is %d years old\n", name, age)
}
```

## Functions with Return Values

Functions can send a value back to the caller using the `return` keyword:

```go
func double(n int) int {
    return n * 2
}
```

The `int` after the parentheses is the **return type**. It tells Go that this function will return an integer.

You use the returned value like this:

```go
result := double(5)
fmt.Println(result) // 10
```

You can also use the return value directly:

```go
fmt.Println(double(7)) // 14
```

## The Full Function Syntax

Here is the complete structure of a Go function:

```go
func functionName(param1 type1, param2 type2) returnType {
    // function body
    return someValue
}
```

Let me break it down piece by piece:

- **`func`** - The keyword that tells Go you are defining a function
- **`functionName`** - The name of your function (how you call it)
- **`(param1 type1, param2 type2)`** - The parameter list (inputs)
- **`returnType`** - What type of value the function returns
- **`{ }`** - The function body (the code that runs)
- **`return`** - Sends a value back to the caller

## Naming Conventions

Go has clear rules about naming functions:

- Use **camelCase** for function names: `calculateTotal`, `processData`, `isValid`
- If the name starts with an **uppercase letter**, the function is **exported** (public, accessible from other packages)
- If the name starts with a **lowercase letter**, the function is **unexported** (private, only accessible within the same package)

```go
func DoSomething() {}  // exported (public)
func doSomething() {}  // unexported (private)
```

This is a big concept in Go. The capitalization is not just style. It is how Go controls visibility. I found this really elegant compared to keywords like `public` and `private` in other languages.

## Package-Level Functions vs Methods

The functions we have been writing are **package-level functions**. They belong to the package and you call them directly by name.

Later, when we learn about **structs**, we will write **methods**. These are functions that belong to a specific type:

```go
// Package-level function
func greet(name string) {
    fmt.Println("Hello,", name)
}

// Method (attached to a type)
// We will learn about this later
func (u User) greet() {
    fmt.Println("Hello,", u.Name)
}
```

For now, just know that methods exist and we will cover them in detail when we get to structs.

## A Complete Example

Here is a program with several functions working together:

```go
package main

import "fmt"

// Function with no parameters, no return
func showMenu() {
    fmt.Println("=== Calculator ===")
    fmt.Println("1. Add")
    fmt.Println("2. Subtract")
    fmt.Println("3. Multiply")
    fmt.Println("4. Divide")
}

// Function with parameters and a return value
func add(a, b int) int {
    return a + b
}

func subtract(a, b int) int {
    return a - b
}

func multiply(a, b int) int {
    return a * b
}

func main() {
    showMenu()

    x := 10
    y := 3

    fmt.Println("Add:", add(x, y))       // 13
    fmt.Println("Subtract:", subtract(x, y)) // 7
    fmt.Println("Multiply:", multiply(x, y)) // 30
}
```

Notice how each function does **one thing**. `add` only adds. `showMenu` only shows the menu. This makes the code easy to read and easy to test.

## Function Best Practices I Am Learning

As I practice writing functions, I am trying to follow these guidelines:

**One function, one job.** A function should do exactly one thing. If you find a function doing multiple things, split it into smaller functions.

**Keep functions short.** If a function is getting long (more than 20-30 lines), it probably needs to be broken up.

**Use descriptive names.** `calculateTotal` is better than `calc`. `isEven` is better than `check`. The name should tell you what the function does.

**Order matters.** In Go, the order you define functions does not matter. You can call a function before defining it in the file. The compiler handles it.

```go
func main() {
    greet() // works even though greet is defined below
}

func greet() {
    fmt.Println("Hello!")
}
```

## What is Next?

Now that we understand the basics of functions, let us dive deeper into **return values**. Go has a unique feature: functions can return multiple values. This is something that really surprised me when I first saw it, and it changes how you write and think about code. Let us explore that next.
