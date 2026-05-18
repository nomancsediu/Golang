# Pointers in Go

## What Is a Pointer?

A **pointer** is a variable that stores the **memory address** of another variable. Instead of holding a value directly, a pointer holds the location where the value lives in memory.

Think of it like this: if a variable is a house, a pointer is the address of that house. You can use the address to find the house and see what is inside.

Pointers scared me when I first encountered them in C. Memory addresses, dereferencing, segmentation faults. It felt dangerous. But Go pointers are much simpler and safer. Let me show you.

## Creating a Pointer with &

The `&` operator (called the **address-of operator**) gives you the memory address of a variable:

```go
x := 42
p := &x  // p is a pointer to x, type is *int

fmt.Println(x)  // 42
fmt.Println(p)  // 0xc0000b6010 (some memory address)
fmt.Println(&x) // 0xc0000b6010 (same address as p)
```

`p` holds the address of `x`. The type of `p` is `*int`, which you read as "pointer to int."

## Dereferencing with *

The `*` operator (called the **dereferencing operator**) lets you read or modify the value at the address the pointer points to:

```go
x := 42
p := &x

fmt.Println(*p)  // 42 - read the value at the address
*p = 100         // modify the value at the address
fmt.Println(x)   // 100 - x was changed through the pointer
```

This is the power of pointers: you can modify a variable from a different scope by passing its address to a function.

## Zero Value of a Pointer

The zero value of a pointer is **nil**. A nil pointer does not point to anything:

```go
var p *int
fmt.Println(p) // <nil>
```

Trying to dereference a nil pointer causes a **panic** (runtime error). This is one of the most common pointer mistakes:

```go
var p *int
fmt.Println(*p) // PANIC: invalid memory address or nil pointer dereference
```

Always check if a pointer is nil before dereferencing if there is any chance it could be nil:

```go
if p != nil {
    fmt.Println(*p) // safe
}
```

## Pointers to Structs

Pointers to structs are extremely common in Go. You will use them all the time:

```go
type Person struct {
    Name string
    Age  int
}

alice := &Person{Name: "Alice", Age: 30}

// Access fields without explicit dereferencing
fmt.Println(alice.Name) // "Alice" - Go auto-dereferences
alice.Age = 31           // same: Go auto-dereferences
```

Go automatically dereferences struct pointers when you access fields. You do not need to write `(*alice).Name`. Just `alice.Name` works. This makes code cleaner and easier to read.

## Pointer Arithmetic: Not Allowed in Go

This is a big difference from C. In C, you can do pointer arithmetic like `p++` to move to the next memory address. **Go does not allow pointer arithmetic.** This is by design. It prevents a whole category of bugs and security vulnerabilities.

```go
x := [3]int{1, 2, 3}
p := &x[0]
// p++       // COMPILE ERROR: invalid operation
// p = p + 1 // COMPILE ERROR: invalid operation
```

You cannot increment a pointer, add offsets to it, or perform any arithmetic on it. The only things you can do with a pointer are:

- Get the address of a variable with `&`
- Dereference a pointer with `*`
- Compare a pointer to `nil`

That is it. Simple and safe.

## The new() Function

Go has a built-in `new()` function that allocates memory for a type, sets it to the zero value, and returns a pointer:

```go
p := new(int)     // p is *int, points to an int with value 0
fmt.Println(*p)   // 0
*p = 42
fmt.Println(*p)   // 42

s := new(Person)  // s is *Person
fmt.Println(s.Name) // ""
fmt.Println(s.Age)  // 0
```

In practice, I rarely use `new()`. I prefer struct literals with `&` because they are more explicit:

```go
// I prefer this:
p := &Person{Name: "Alice", Age: 30}

// Over this:
p := new(Person)
p.Name = "Alice"
p.Age = 30
```

## When to Use Pointers

There are three main reasons to use pointers:

**1. Modify the original value.** If you want a function to change a variable, you must pass a pointer:

```go
func increment(n *int) {
    *n++
}

func main() {
    x := 5
    increment(&x)
    fmt.Println(x) // 6
}
```

Without the pointer, `increment` would only modify a copy of `x`.

**2. Avoid copying large data.** If you have a large struct (say, a struct with a big slice or many fields), passing it by value copies all that data. A pointer avoids the copy:

```go
type BigData struct {
    Records [1000000]int
    Name    string
}

// Bad: copies the entire struct
func processValue(data BigData) { /* ... */ }

// Good: only copies the pointer (8 bytes)
func processPointer(data *BigData) { /* ... */ }
```

**3. Implement interfaces with pointer receivers.** If your type has methods with pointer receivers, you usually need to pass a pointer to satisfy an interface:

```go
type Writer interface {
    Write(data []byte)
}

type FileWriter struct {
    // ... fields
}

func (f *FileWriter) Write(data []byte) { /* ... */ }

// Only *FileWriter satisfies Writer, not FileWriter
var w Writer = &FileWriter{}
```

## Common Mistakes

**1. Dereferencing a nil pointer:**

```go
var p *int
*p = 5 // PANIC!
```

Always check for nil or make sure the pointer is initialized.

**2. Returning a pointer to a local variable:**

This is actually fine in Go! Unlike C, Go's compiler detects when a local variable's address escapes the function and allocates it on the heap instead of the stack:

```go
func createPerson() *Person {
    p := Person{Name: "Alice", Age: 30}
    return &p // This is safe in Go
}
```

In C, this would be a bug (returning a dangling pointer to stack memory). In Go, the compiler handles it for you. This is called **escape analysis**.

**3. Confusing * in types vs expressions:**

- In a type: `*int` means "pointer to int"
- In an expression: `*p` means "get the value at the pointer"
- In an expression: `&x` means "get the address of x"

The `*` symbol does double duty, which can be confusing at first. Just remember: in types, `*` means "pointer to." In expressions, `*` means "dereference."

## A Personal Note

I used to avoid pointers because they felt scary and C-like. But in Go, pointers are simple. You cannot do arithmetic on them. You cannot accidentally corrupt memory. The worst that can happen is a nil pointer dereference, which gives a clear panic instead of silent corruption.

Now I use pointers naturally. When I want to modify something, I pass a pointer. When I have a big struct, I pass a pointer. When I need to satisfy an interface, I use a pointer. It just becomes second nature.

Go pointers are C pointers with the dangerous parts removed. They give you the power without the footguns. That is a good trade-off.
