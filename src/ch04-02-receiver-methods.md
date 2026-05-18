# Receiver Functions / Methods in Go

## What Are Methods?

A **method** in Go is just a function with a special argument called a **receiver**. The receiver appears before the function name, in parentheses. It attaches the function to a specific type, so you can call it using dot notation like `person.Greet()`.

If you come from another language, a method in Go is similar to a method on a class. But in Go, the method is not defined inside the struct. It is a separate function that happens to take the struct as its first argument.

```go
type Person struct {
    Name string
    Age  int
}

// This is a method with a value receiver
func (p Person) Greet() string {
    return "Hello, my name is " + p.Name
}

func main() {
    alice := Person{Name: "Alice", Age: 30}
    fmt.Println(alice.Greet()) // "Hello, my name is Alice"
}
```

See that `(p Person)` before the function name? That is the receiver. It says "this method belongs to the Person type, and inside the method, `p` refers to the instance."

## Value Receiver vs Pointer Receiver

This is one of the most important concepts in Go. There are two kinds of receivers:

### Value Receiver

```go
func (p Person) Greet() string {
    return "Hello, my name is " + p.Name
}
```

With a **value receiver**, the method operates on a **copy** of the struct. Any changes you make inside the method do not affect the original. This is just like passing a value to a function.

```go
func (p Person) HaveBirthday() {
    p.Age++ // this changes the COPY, not the original
}

func main() {
    alice := Person{Name: "Alice", Age: 30}
    alice.HaveBirthday()
    fmt.Println(alice.Age) // still 30!
}
```

The `Age` does not change because `HaveBirthday` modified a copy of `alice`, not `alice` itself.

### Pointer Receiver

```go
func (p *Person) HaveBirthday() {
    p.Age++ // this changes the ORIGINAL
}

func main() {
    alice := Person{Name: "Alice", Age: 30}
    alice.HaveBirthday()
    fmt.Println(alice.Age) // 31! The original was modified
}
```

With a **pointer receiver**, the method receives a **pointer** to the original struct. Any changes you make inside the method affect the original. This is how you modify data through methods.

Notice that we called `alice.HaveBirthday()` even though `alice` is a value, not a pointer. Go automatically takes the address (`&alice`) for you. The same convenience works the other way too: if you have a pointer and call a value receiver method, Go automatically dereferences it.

## When to Use Pointer Receiver

There are three main reasons to choose a pointer receiver:

**1. You need to modify the receiver.** If your method changes the struct's data, you must use a pointer receiver. A value receiver would only modify a copy.

**2. The struct is large.** If your struct has many fields or contains large data (like a big slice or map), copying it on every method call is wasteful. A pointer receiver avoids the copy.

**3. Consistency.** If some methods on a type need pointer receivers (because they modify data), it is a good practice to make all methods on that type use pointer receivers. This way, the caller does not have to think about which methods use which receiver type.

A common Go guideline: **when in doubt, use a pointer receiver.**

## Method Set Rules

This is a subtle but important topic. The set of methods you can call on a value depends on whether you have a value or a pointer:

- A **value** of type `T` can only call methods with **value receivers**.
- A **pointer** of type `*T` can call methods with **both value and pointer receivers**.

```go
type Person struct {
    Name string
}

func (p Person)   Greet()  string { return "Hi, " + p.Name }
func (p *Person)  Update(name string) { p.Name = name }

func main() {
    v := Person{Name: "Alice"}
    p := &Person{Name: "Bob"}

    v.Greet()       // OK: value can call value receiver
    v.Update("New") // OK: Go automatically takes address (&v)
    p.Greet()       // OK: pointer can call value receiver
    p.Update("New") // OK: pointer can call pointer receiver
}
```

In practice, Go makes this very convenient by auto-taking addresses and auto-dereferencing. But the rule matters when implementing interfaces. If an interface requires a method with a pointer receiver, only a pointer to your type satisfies that interface, not the value.

## Methods on Any Named Type

Methods are not limited to structs. You can define methods on any **named type** in your package:

```go
type MyString string

func (s MyString) Shout() string {
    return strings.ToUpper(string(s)) + "!!!"
}

type Celsius float64

func (c Celsius) ToFahrenheit() float64 {
    return float64(c)*9/5 + 32
}

func main() {
    msg := MyString("hello")
    fmt.Println(msg.Shout()) // "HELLO!!!"

    temp := Celsius(100)
    fmt.Println(temp.ToFahrenheit()) // 212
}
```

You cannot define methods on types from other packages (like `int` or `string` directly). But you can create your own named type based on them and add methods. This is a powerful pattern.

## Chaining Methods

You can chain method calls by having each method return the receiver:

```go
type Builder struct {
    data string
}

func (b *Builder) Add(s string) *Builder {
    b.data += s
    return b
}

func main() {
    result := &Builder{}.
        Add("Hello").
        Add(", ").
        Add("World")
    fmt.Println(result.data) // "Hello, World"
}
```

This pattern is common in builder-style APIs. Each method returns the pointer so the next method can be called on it.

## A Personal Note

When I first saw receiver methods, I thought "oh, these are just like class methods." And they kind of are. But the difference is that in Go, the method is not trapped inside a class definition. It is a free function that happens to be associated with a type. You can add methods to existing types. You can organize your methods across multiple files. It feels more flexible.

The value vs pointer receiver thing confused me for a while. My rule of thumb now: if the method modifies the receiver, use pointer. If the struct is big, use pointer. Otherwise, either works, but be consistent across all methods on the same type.

Methods are what make Go structs feel like objects. You have data (struct) and behavior (methods) working together, but without the heavy machinery of classes and inheritance. It is OOP, but stripped down to its essential parts.
