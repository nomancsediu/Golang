# Bogus Data Types in Go

## What Are Bogus / Phantom Types?

The term **bogus data type** (also called **phantom type**) refers to a type that exists at compile time but has no different runtime representation compared to its underlying type. In Go, this happens when you create a **new named type** based on an existing type. The new type is treated as distinct by the compiler, but at runtime, it is stored and behaves just like the original type.

This is a powerful idea. It lets you use the type system to catch mistakes at compile time, even though the underlying data is the same.

## Type Alias vs Type Definition

Go has two ways to create a new type from an existing one, and they behave very differently:

### Type Alias (with =)

```go
type MyInt = int
```

A **type alias** creates another name for the same type. `MyInt` and `int` are completely interchangeable. They are the same type:

```go
type MyInt = int

var x MyInt = 42
var y int = 100

x = y // OK: same type
y = x // OK: same type
fmt.Println(x + y) // 142
```

You can use `MyInt` anywhere you use `int` and vice versa. There is no type safety benefit here. It is just a different name for readability or backward compatibility.

### Type Definition (without =)

```go
type MyInt int
```

A **type definition** creates a brand new, distinct type. `MyInt` and `int` are **different types**, even though `MyInt` has the same underlying representation as `int`:

```go
type MyInt int

var x MyInt = 42
var y int = 100

x = y // COMPILE ERROR: cannot use y (int) as MyInt
y = x // COMPILE ERROR: cannot use x (MyInt) as int
```

This is the key difference. The compiler treats them as separate types. You need an explicit conversion to go between them:

```go
x = MyInt(y)  // explicit conversion from int to MyInt
y = int(x)    // explicit conversion from MyInt to int
```

This is what makes it a "bogus" or "phantom" type. The runtime representation is the same as `int`, but the compiler acts as if it is something entirely different. The distinction only exists at compile time.

## Using Type Definitions for Type Safety

This is where bogus types shine. You can use them to prevent mixing up different kinds of values that happen to share the same underlying type:

### Type-Safe IDs

Imagine you have a system with users and orders. Both have integer IDs. Without type definitions, you might accidentally pass a user ID where an order ID is expected:

```go
// BAD: both IDs are just ints
func getUser(userID int) { /* ... */ }
func getOrder(orderID int) { /* ... */ }

var uid int = 42
var oid int = 7

getUser(oid) // oops: passed order ID as user ID, compiles fine!
```

With type definitions, the compiler catches this mistake:

```go
type UserID int
type OrderID int

func getUser(id UserID) { /* ... */ }
func getOrder(id OrderID) { /* ... */ }

var uid UserID = 42
var oid OrderID = 7

getUser(oid) // COMPILE ERROR: cannot use oid (OrderID) as UserID
getUser(uid) // OK
```

Now you cannot accidentally mix up the two kinds of IDs. The type system enforces correctness at compile time, for free.

### Type-Safe String Types

The same pattern works with strings:

```go
type Email string
type Phone string

func sendEmail(addr Email) { /* ... */ }
func callNumber(num Phone) { /* ... */ }

var e Email = "alice@example.com"
var p Phone = "+1234567890"

sendEmail(p) // COMPILE ERROR: cannot use p (Phone) as Email
sendEmail(e) // OK
```

This prevents you from passing a phone number to a function that expects an email address.

## Adding Methods to Bogus Types

Since a type definition creates a new named type, you can attach methods to it. This is very useful:

```go
type Celsius float64
type Fahrenheit float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((float64(f) - 32) * 5 / 9)
}

func main() {
    boiling := Celsius(100)
    fmt.Println(boiling.ToFahrenheit()) // 212
}
```

You cannot add methods to `float64` directly, but you can add them to `Celsius` because it is your own named type. This gives you type safety and methods in one package.

## Preventing Invalid Values at Compile Time

You can use bogus types to make invalid states unrepresentable. Here is a common pattern:

```go
// Raw status string: anything goes
status := "pending"    // OK
status := "shipped"    // OK
status := "bananas"    // Also OK! But "bananas" is not a valid status

// Type-safe status
type OrderStatus string

const (
    StatusPending OrderStatus = "pending"
    StatusShipped OrderStatus = "shipped"
    StatusDelivered OrderStatus = "delivered"
)

func updateStatus(s OrderStatus) { /* ... */ }

updateStatus(StatusPending) // OK
updateStatus(StatusShipped) // OK
updateStatus("bananas")     // COMPILE ERROR: cannot use "bananas" as OrderStatus
```

Well, technically the last line does compile because `"bananas"` is a constant that can be converted. But when you use the constants you defined, it guides developers toward valid values.

A stronger approach uses a struct with unexported fields:

```go
type OrderStatus struct {
    value string
}

var (
    StatusPending   = OrderStatus{"pending"}
    StatusShipped   = OrderStatus{"shipped"}
    StatusDelivered = OrderStatus{"delivered"}
)

func updateStatus(s OrderStatus) {
    fmt.Println(s.value)
}

updateStatus(StatusPending) // OK
// No way to create an arbitrary OrderStatus from outside the package
// because the `value` field is unexported
```

## Common Uses in Real Code

Here are some patterns I see in real Go codebases:

**1. Database IDs:**

```go
type UserID string
type PostID string
type CommentID string
```

**2. Domain-specific units:**

```go
type Miles float64
type Kilometers float64
type Pounds float64
type Kilograms float64
```

**3. API parameter types:**

```go
type APIKey string
type AccessToken string
type RefreshToken string
```

**4. Configuration values:**

```go
type Port int
type Timeout int // in milliseconds
```

## A Personal Note

I did not appreciate bogus types at first. "Why create a new type that is just an int?" I thought. "Just use int." But after spending time in a codebase where everything was `int` and `string`, I started seeing bugs that bogus types would have prevented. Passing a user ID where a post ID was expected. Mixing up milliseconds and seconds. Confusing a token with a key.

Bogus types cost almost nothing. They are just a name. But they give the compiler extra information to catch your mistakes. And the best bugs are the ones the compiler catches before your code ever runs.

Now I use type definitions liberally. Any time I have a concept that happens to be represented by a built-in type but has a specific meaning, I give it its own type. It is a small habit that makes a big difference.
