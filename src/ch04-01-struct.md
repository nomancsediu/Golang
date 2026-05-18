# Struct in Go

## What Is a Struct?

A **struct** is a collection of named fields. It is Go's way of grouping related data together into one unit. If you come from an object-oriented language like Java or Python, think of a struct as a class that only has data fields and no methods. (We will add methods later with receiver functions.)

I like to think of a struct as a form. Each field is a blank on the form, and when you create a struct instance, you fill in the blanks.

## Defining a Struct

You define a struct using the `type` keyword:

```go
type Person struct {
    Name string
    Age  int
}
```

That is it. No constructor, no `new` keyword required, no ceremony. You just say "I want a type called Person with a Name field that is a string and an Age field that is an int."

By convention, struct field names start with an **uppercase letter** if you want them to be **exported** (accessible from other packages). If a field starts with a lowercase letter, it is **unexported** and only visible within the same package.

```go
type Person struct {
    Name    string  // exported, visible everywhere
    Age     int     // exported
    address string  // unexported, only visible in this package
}
```

## Creating Instances

There are several ways to create a struct instance:

**1. Struct literal with field names (most common):**

```go
p1 := Person{
    Name: "Alice",
    Age:  30,
}
```

I prefer this style because it is explicit and self-documenting. If someone reads your code, they can see exactly which field gets which value.

**2. Struct literal without field names (positional):**

```go
p2 := Person{"Bob", 25}
```

This works but I avoid it. If you add a new field to the struct later, this code breaks. Use field names instead.

**3. Using the new() function:**

```go
p3 := new(Person)
// p3 is *Person (a pointer), all fields are zero values
fmt.Println(p3.Name) // ""
fmt.Println(p3.Age)  // 0
```

`new()` allocates memory for the struct, sets all fields to their **zero values**, and returns a pointer to it.

**4. Using the address-of operator &:**

```go
p4 := &Person{
    Name: "Diana",
    Age:  28,
}
// p4 is *Person (a pointer to Person)
```

This is the most common way to get a pointer to a struct. I use this all the time.

## Accessing Fields

You use **dot notation** to access and modify fields. This works on both values and pointers. Go automatically dereferences pointers for you:

```go
p := Person{Name: "Alice", Age: 30}
fmt.Println(p.Name) // "Alice"
p.Age = 31          // modify a field

pp := &p
pp.Name = "Alicia"  // Go dereferences the pointer automatically
fmt.Println(p.Name) // "Alicia"
```

Notice that last line. Even though `pp` is a pointer, you do not need to write `(*pp).Name`. Go handles the dereferencing for you. This is a small but wonderful convenience.

## Nested Structs

A struct can contain another struct as a field:

```go
type Address struct {
    City    string
    Country string
}

type Employee struct {
    Name    string
    Address Address  // nested struct field
    Salary  float64
}

e := Employee{
    Name: "Alice",
    Address: Address{
        City:    "Berlin",
        Country: "Germany",
    },
    Salary: 50000,
}

fmt.Println(e.Address.City) // "Berlin"
```

You chain the dot notation to access nested fields. It reads naturally: "employee's address's city."

## Anonymous Fields (Embedding)

Go has a feature called **embedding**, where you can put a struct inside another struct without giving it a field name:

```go
type Animal struct {
    Name string
}

type Dog struct {
    Animal    // embedded struct, no field name
    Breed string
}

d := Dog{
    Animal: Animal{Name: "Rex"},
    Breed:  "Labrador",
}
fmt.Println(d.Name)   // "Rex" - direct access to embedded field
fmt.Println(d.Animal.Name) // "Rex" - explicit access also works
```

This is Go's alternative to inheritance. The inner struct's fields are "promoted" to the outer struct. You can access them directly without going through the embedded type name.

This is not inheritance though. It is **composition**. The Dog struct has an Animal, it does not inherit from Animal. This distinction matters when you think about design.

## Struct Tags

Struct tags are extra metadata attached to fields. They are used heavily by packages like `encoding/json` and database ORMs:

```go
type User struct {
    ID       int    `json:"id" db:"user_id"`
    Username string `json:"username" db:"user_name"`
    Password string `json:"-" db:"-"`        // "-" means skip this field
    Email    string `json:"email,omitempty"` // omit if empty
}
```

When you marshal this struct to JSON, the tags control the output:

```go
u := User{ID: 1, Username: "alice", Password: "secret", Email: ""}
data, _ := json.Marshal(u)
fmt.Println(string(data))
// {"id":1,"username":"alice"}
// Password is skipped because of "-"
// Email is omitted because it is empty and has "omitempty"
```

I use struct tags all the time when building APIs. They are simple but incredibly useful.

## Structs Are Value Types

This is important: **structs are value types**. When you assign a struct to a new variable or pass it to a function, Go makes a copy:

```go
p1 := Person{Name: "Alice", Age: 30}
p2 := p1       // p2 is a COPY of p1
p2.Name = "Bob"
fmt.Println(p1.Name) // "Alice" - original unchanged
fmt.Println(p2.Name) // "Bob"   - only the copy changed
```

If you want two variables to share the same struct, you need to use a pointer:

```go
p1 := &Person{Name: "Alice", Age: 30}
p2 := p1        // both point to the same struct
p2.Name = "Bob"
fmt.Println(p1.Name) // "Bob" - both see the change
```

Understanding the difference between value and pointer semantics is one of the most important things in Go. We will go deeper into this when we cover pointers.

## A Personal Note

Coming from Python and Java, I used to miss classes. Where are my constructors? Where is my inheritance? How do I organize code without a class hierarchy?

After a few weeks of writing Go, I realized that structs are enough. They are simpler, they compose better than inheritance, and they avoid the deep hierarchies that make OOP code hard to change. A struct with a few methods can do everything a class does, with less complexity.

Go's philosophy is: **prefer composition over inheritance, and keep it simple.** Structs embody that perfectly.
