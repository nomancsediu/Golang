# Variables and Data Types in Go

## Storing Data: The Foundation of Every Program

Every program needs to store data. A user's name, a score, a temperature reading, whether a switch is on or off. In Go, we use **variables** to store data, and every variable has a **type** that tells Go what kind of data it holds. Let me share what I have learned about how Go handles this.

## Declaring Variables

Go gives you two main ways to declare a variable. The first is the `var` keyword:

```go
var name string = "Noman"
var age int = 25
var isLearning bool = true
```

With `var`, you explicitly declare the variable name, its type, and its value. This is clear and explicit. You can also declare without an initial value:

```go
var city string
var population int
```

When you declare a variable without assigning a value, Go gives it a **zero value** (more on that below).

## The Short Declaration Operator

The second way is the **short declaration operator** `:=`. This is the one I use most often:

```go
name := "Noman"
age := 25
isLearning := true
price := 19.99
```

With `:=`, Go **infers** the type automatically from the value you assign. You do not need to write the type explicitly. Go figures it out.

This is cleaner and more concise. But there is a catch: **you can only use `:=` inside a function**. You cannot use it at the package level (outside of functions).

## When to Use `var` vs `:=`

This confused me at first, so here is the simple rule I follow:

- Use **`:=`** when you are inside a function and have an initial value (most common case)
- Use **`var`** when you need to declare a variable without an initial value
- Use **`var`** when you are at the package level (outside any function)
- Use **`var`** when you want to be explicit about the type

```go
package main

import "fmt"

// Package-level: must use var
var version string = "1.0.0"

func main() {
    // Inside function: can use :=
    name := "Noman"
    fmt.Println(name, version)

    // Need zero value: use var
    var score int
    fmt.Println(score) // prints 0
}
```

## Basic Data Types

Go has several built-in data types. Here are the ones I use most often:

### String

A **string** is a sequence of characters, enclosed in double quotes.

```go
greeting := "Hello, Go!"
name := "Noman"
empty := "" // empty string
```

Strings in Go are immutable. Once created, you cannot change individual characters. You can create a new string based on an existing one, but the original stays the same.

### Integer Types

Go has many integer types, which was surprising to me at first:

```go
var a int     // platform-dependent size, usually 64-bit on modern systems
var b int8    // -128 to 127
var c int16   // -32768 to 32767
var d int32   // -2147483648 to 2147483647
var e int64   // very large range

var f uint    // unsigned, platform-dependent
var g uint8   // 0 to 255
var h uint16  // 0 to 65535
var i uint32  // 0 to 4294967295
var j uint64  // very large range
```

For most cases, just use **`int`**. The specific sizes are for when you need precise control over memory, like working with binary protocols or embedded systems.

### Float Types

```go
var pi float32 = 3.14
var precise float64 = 3.141592653589793
```

**`float64`** is the default when you use `:=` with a decimal number. It gives you more precision, so use it unless you have a specific reason to use `float32`.

```go
temperature := 36.6 // Go infers float64
```

### Boolean

```go
var isActive bool = true
var isComplete bool = false
```

A **bool** can only be `true` or `false`. It is used for conditions and logic. Simple and straightforward.

## Zero Values

This is a concept I really like in Go. When you declare a variable without assigning a value, Go does not give you garbage or null. It gives you a sensible **zero value**:

```go
package main

import "fmt"

func main() {
    var n int
    var s string
    var b bool
    var f float64

    fmt.Println(n) // 0
    fmt.Println(s) // "" (empty string)
    fmt.Println(b) // false
    fmt.Println(f) // 0
}
```

| Type     | Zero Value |
|----------|------------|
| int      | 0          |
| float64  | 0          |
| string   | ""         |
| bool     | false      |

This means there are no "undefined" surprises. Every variable always has a valid value.

## Constants

When you have a value that should never change, use a **constant** with the `const` keyword:

```go
const pi = 3.14159
const appName = "Go Mastery"
const maxRetries = 3
```

Constants cannot be changed after declaration. If you try, the compiler will give you an error. They must be known at compile time, so you cannot use `:=` with constants.

```go
const daysInWeek = 7
// daysInWeek = 8 // ERROR: cannot assign to daysInWeek
```

I use constants for things like configuration values, mathematical constants, and fixed strings that appear throughout my code.

## Type Conversion: No Shortcuts Allowed

This is one of the strictest things about Go, and it took me a while to get used to. **Go does not do implicit type conversion.** Ever. If you have an `int` and need a `float64`, you must convert it explicitly:

```go
package main

import "fmt"

func main() {
    var x int = 42
    var y float64 = float64(x) // explicit conversion
    var z int = int(y)         // converting back

    fmt.Println(x) // 42
    fmt.Println(y) // 42
    fmt.Println(z) // 42

    // This would cause a compiler error:
    // var wrong float64 = x // cannot use x (int) as float64
}
```

Same with strings and numbers. You cannot just concatenate a number to a string:

```go
age := 25
// message := "I am " + age // ERROR!
message := "I am " + fmt.Sprintf("%d", age) // correct way
fmt.Println(message)
```

You need to convert the number to a string first using `fmt.Sprintf()` or `strconv.Itoa()`.

```go
import "strconv"

age := 25
message := "I am " + strconv.Itoa(age)
fmt.Println(message)
```

## My Thoughts on Go's Strict Typing

At first, I found Go's type system annoying. In Python, you can just write `x = 42` and later `x = "hello"` and nobody cares. In JavaScript, `"5" + 3` gives you `"53"` (weird, but it works). Go does not allow any of that.

But now I see the wisdom. **Strict typing prevents bugs.** If a function expects an `int`, you cannot accidentally pass it a `string`. The compiler catches these mistakes before your code ever runs. It saves debugging time. It saves production incidents.

The tradeoff is simple: you write a little more code upfront, but you spend a lot less time fixing bugs later. I think that is a good deal.

## Quick Reference

Here is a cheat sheet I keep handy:

```go
// Declaration styles
var name string = "Noman"   // full declaration
var name = "Noman"          // type inferred
name := "Noman"             // short declaration (functions only)

// Common types
s := "hello"    // string
n := 42         // int
f := 3.14       // float64
b := true       // bool

// Constants
const pi = 3.14159

// Type conversion
x := int(3.14)       // float64 to int = 3
y := float64(42)     // int to float64 = 42.0
s := strconv.Itoa(42) // int to string = "42"
```

Now that we know how to store data, let us learn how to make decisions with **if, else, and switch**.
