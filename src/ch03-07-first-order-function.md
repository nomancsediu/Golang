# Ch03 07 First Order Function

A **first-order function** is the simplest kind of function. It takes data in, it returns data out. No functions as inputs, no functions as outputs. Just plain values going in and plain values coming out.

## What Is a First-Order Function?

A first-order function is a function that:
- Accepts only **data types** as parameters (like `int`, `string`, `bool`, `struct`)
- Returns only **data types** as results
- Does **not** take a function as a parameter
- Does **not** return a function

Most of the functions you write are first-order functions. They are simple, predictable, and easy to understand.

```go
// This is a first-order function
// It takes data (int) and returns data (int)
func double(n int) int {
    return n * 2
}

// This is also a first-order function
// It takes data (string) and returns data (bool)
func isEmpty(s string) bool {
    return len(s) == 0
}
```

## Examples of First-Order Functions

Here are some common first-order functions:

```go
// Mathematical operations
func add(a, b int) int {
    return a + b
}

// String operations
func toUpper(s string) string {
    return strings.ToUpper(s)
}

// Validation
func isValidAge(age int) bool {
    return age >= 0 && age <= 150
}

// Data transformation
func celsiusToFahrenheit(c float64) float64 {
    return c*9/5 + 32
}

// Struct operations
type User struct {
    Name string
    Age  int
}

func createUser(name string, age int) User {
    return User{Name: name, Age: age}
}

func isAdult(u User) bool {
    return u.Age >= 18
}
```

All of these take data in and return data out. No functions involved anywhere.

## When to Use First-Order Functions

First-order functions are the right choice in most situations. You should use them when:

**The operation is straightforward.** If a function does one clear thing with data, there is no need to complicate it with function parameters.

**You do not need flexibility.** If the logic is always the same, a first-order function is perfect. `double(n)` always doubles. It does not need a parameter to decide how to process the number.

**You want predictability.** First-order functions are easy to reason about. You know exactly what goes in and what comes out. No surprises.

**You are writing utility functions.** Things like `len()`, `cap()`, `append()` are all first-order. They are simple tools that do one job.

## Comparison with Higher-Order Functions

To understand first-order functions better, it helps to compare them with **higher-order functions**. A higher-order function takes a function as a parameter or returns a function.

```go
// FIRST-ORDER: takes data only
func doubleAll(nums []int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = n * 2
    }
    return result
}

// HIGHER-ORDER: takes a function as parameter
func mapInts(nums []int, transform func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = transform(n)
    }
    return result
}
```

The first function always doubles. The second function can do anything because you pass the transformation logic as an argument.

```go
func main() {
    numbers := []int{1, 2, 3, 4, 5}

    // First-order: only doubles
    doubled := doubleAll(numbers)
    fmt.Println(doubled) // [2 4 6 8 10]

    // Higher-order: flexible
    tripled := mapInts(numbers, func(n int) int {
        return n * 3
    })
    fmt.Println(tripled) // [3 6 9 12 15]

    squared := mapInts(numbers, func(n int) int {
        return n * n
    })
    fmt.Println(squared) // [1 4 9 16 25]
}
```

## The Trade-off

| First-Order Function | Higher-Order Function |
|---|---|
| Simple and predictable | Flexible and reusable |
| Easy to understand | Can be harder to read |
| Does one specific thing | Can do many things |
| Less code for one use case | More code upfront, less code overall |
| No function parameters | Takes or returns functions |

## When First-Order Is Not Enough

Sometimes a first-order function is too rigid. If you find yourself writing multiple functions that do almost the same thing, it might be time for a higher-order function:

```go
// Too many similar first-order functions
func doubleAll(nums []int) []int { ... }
func tripleAll(nums []int) []int { ... }
func squareAll(nums []int) []int { ... }
func negateAll(nums []int) []int { ... }

// One higher-order function replaces them all
func mapInts(nums []int, transform func(int) int) []int { ... }
```

But do not over-engineer. If you only need `doubleAll`, write `doubleAll`. Reach for higher-order functions when you actually need the flexibility.

## Personal Note

Most of the functions I write are first-order functions. And that is fine. There is nothing wrong with simple. I used to think that using higher-order functions everywhere made my code more "advanced." It did not. It made it harder to read. First-order functions are the bread and butter of programming. Use them until you genuinely need more flexibility, then reach for higher-order functions. The next chapter covers higher-order functions in detail.
