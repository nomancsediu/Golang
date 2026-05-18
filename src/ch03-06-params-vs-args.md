# Ch03 06 Params vs Args

This topic confused me for a long time. People use "parameter" and "argument" interchangeably, but they are different things. Once I understood the difference, it made reading documentation and error messages much easier.

## Parameter vs Argument: What Is the Difference?

A **parameter** is the variable in the function definition. It is a placeholder. Think of it as an empty box with a label on it.

An **argument** is the actual value you pass when you call the function. It is what you put in the box.

```go
// "name" is the PARAMETER
func greet(name string) {
    fmt.Println("Hello,", name)
}

// "Alice" is the ARGUMENT
greet("Alice")
```

That is it. Parameters are in the definition. Arguments are in the call. Simple, but important.

Another way to think about it:

- **Parameter** = the parking spot
- **Argument** = the car you park in it

## Types of Parameters

Go has a few different kinds of parameters. Each one behaves differently.

### Regular Parameters

These are the standard parameters you already know:

```go
func add(a int, b int) int {
    return a + b
}

func main() {
    result := add(3, 5) // 3 and 5 are arguments
    fmt.Println(result) // 8
}
```

You can also group parameters of the same type:

```go
func add(a, b int) int {
    return a + b
}
```

Both `a` and `b` are `int`. The type only needs to be written once at the end.

### Variadic Parameters

A **variadic parameter** accepts zero or more arguments. You define it with three dots (`...`) before the type:

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))       // 6
    fmt.Println(sum(10, 20))        // 30
    fmt.Println(sum())              // 0
    fmt.Println(sum(1, 2, 3, 4, 5)) // 15
}
```

Inside the function, `nums` is a slice of `int`. You can iterate over it, check its length, and do anything you would do with a slice.

### Variadic Parameters Must Be Last

You can only have one variadic parameter, and it must be the last parameter:

```go
// VALID: variadic is last
func printFormatted(prefix string, nums ...int) {
    fmt.Print(prefix)
    for _, n := range nums {
        fmt.Print(n, " ")
    }
    fmt.Println()
}

// INVALID: variadic must be last
// func bad(nums ...int, suffix string) { } // compile error
```

This makes sense because otherwise the compiler would not know where the variadic arguments end and the next parameter begins.

### Passing a Slice to a Variadic Parameter

If you already have a slice, you can pass it to a variadic parameter using the `...` operator:

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}
    result := sum(numbers...) // spread the slice
    fmt.Println(result)       // 15
}
```

The `numbers...` syntax "spreads" the slice into individual arguments. Without the `...`, you would get a compile error because you are passing a slice where individual integers are expected.

### Pointer Parameters

A **pointer parameter** lets the function modify the original value:

```go
func increment(n *int) {
    *n = *n + 1
}

func main() {
    x := 5
    increment(&x)
    fmt.Println(x) // 6
}
```

The parameter `n` is a pointer to an `int`. The argument `&x` is the address of `x`. Inside the function, `*n` dereferences the pointer and modifies the original value.

Without a pointer, Go passes arguments by value (copies them), so the original is not changed:

```go
func incrementNoPointer(n int) {
    n = n + 1 // only changes the local copy
}

func main() {
    x := 5
    incrementNoPointer(x)
    fmt.Println(x) // still 5
}
```

## Multiple Return Values as "Output Parameters"

Go functions can return multiple values. Some people think of these as "output parameters," but they are really just return values:

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result) // Result: 3.3333333333333335
}
```

The `divide` function returns two values: the result and an error. This is the standard Go pattern for error handling.

You can also name the return values:

```go
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = fmt.Errorf("cannot divide by zero")
        return // naked return
    }
    result = a / b
    return // naked return
}
```

Named return values can make the code clearer for short functions, but they can be confusing for longer ones. Use them wisely.

## Common Confusion

Here are the mix-ups I used to make:

1. **Calling a parameter an argument.** Remember: parameter is in the definition, argument is in the call.

2. **Forgetting the `...` when passing a slice to a variadic.** You need `slice...` not just `slice`.

3. **Thinking variadic means "required."** Variadic means zero or more. `sum()` with no arguments is valid.

4. **Thinking pointer parameters are always bad.** They are necessary when you need to modify the original value. The standard library uses them all the time (`json.Unmarshal` takes a pointer, for example).

## Personal Note

I used to say "this function takes three arguments" when I meant "this function takes three parameters." Now I am more careful about the distinction. It matters because when you read Go documentation, it uses these terms precisely. If you know the difference, the docs are clearer. If you mix them up, you might misunderstand what the docs are saying.
