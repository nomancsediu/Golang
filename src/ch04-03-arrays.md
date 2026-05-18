# Arrays in Go

## What Is an Array?

An **array** is a fixed-size collection of elements of the same type. That is it. It has a specific number of slots, and each slot holds a value of the same type. The size is part of the type itself, which means `[5]int` and `[3]int` are completely different types in Go.

Arrays are the foundation of slices (which we will cover next), but you will rarely use arrays directly in real Go code. Understanding them is important though, because slices are built on top of arrays.

## Declaring an Array

You declare an array by specifying the size in brackets, followed by the element type:

```go
var numbers [5]int       // array of 5 integers, all zero-valued
var names [3]string      // array of 3 strings, all ""
var flags [4]bool        // array of 4 bools, all false
```

When you declare an array without initializing it, every element gets the **zero value** of its type. For `int`, that is `0`. For `string`, that is `""`. For `bool`, that is `false`.

## Initializing an Array

**1. Array literal (specify all values):**

```go
colors := [3]string{"red", "green", "blue"}
```

**2. Let the compiler count (use `...`):**

```go
numbers := [...]int{10, 20, 30, 40, 50}
// The compiler sees 5 values, so this is [5]int
```

This is handy when you do not want to count the elements yourself. I use this form more than the explicit size form.

**3. Partial initialization:**

```go
nums := [5]int{1, 2} // [1, 2, 0, 0, 0]
// Unspecified positions get the zero value
```

**4. Specific index initialization:**

```go
arr := [5]int{1: 10, 3: 30} // [0, 10, 0, 30, 0]
// Only indices 1 and 3 are set, rest are zero
```

This is useful for sparse arrays where most elements are zero.

## Accessing Elements

You access array elements using **index notation** with zero-based indexing:

```go
fruits := [3]string{"apple", "banana", "cherry"}

fmt.Println(fruits[0])  // "apple"
fmt.Println(fruits[2])  // "cherry"

fruits[1] = "blueberry" // modify an element
fmt.Println(fruits[1])  // "blueberry"
```

## Length with len()

The built-in `len()` function returns the number of elements in an array:

```go
arr := [5]int{1, 2, 3, 4, 5}
fmt.Println(len(arr)) // 5
```

Since arrays have a fixed size, `len()` always returns the same value for the same array type.

## Arrays Are Values

This is critical: **arrays are value types in Go**. When you assign an array to a new variable or pass it to a function, Go copies the entire array:

```go
a := [3]int{1, 2, 3}
b := a        // b is a COPY of a
b[0] = 99
fmt.Println(a) // [1, 2, 3] - original unchanged
fmt.Println(b) // [99, 2, 3] - only the copy changed
```

This is different from languages like Python or JavaScript where arrays are reference types. In Go, the copy is a deep copy of all elements.

## Passing Arrays to Functions

Because arrays are values, passing an array to a function creates a copy. This has two implications:

**1. Modifications inside the function do not affect the original:**

```go
func modify(arr [3]int) {
    arr[0] = 999
}

func main() {
    a := [3]int{1, 2, 3}
    modify(a)
    fmt.Println(a) // [1, 2, 3] - unchanged
}
```

**2. Copying large arrays is expensive:**

```go
func process(arr [1000000]int) {
    // This copies 1 million integers!
    // Very wasteful if you just need to read the data
}
```

If you want a function to modify the original array or avoid copying a large one, pass a **pointer**:

```go
func modify(arr *[3]int) {
    arr[0] = 999
}

func main() {
    a := [3]int{1, 2, 3}
    modify(&a)
    fmt.Println(a) // [999, 2, 3] - original was modified
}
```

## Limitations of Arrays

Arrays have two big limitations:

**1. Fixed size.** You cannot add or remove elements. The size is baked into the type. A `[5]int` will always have exactly 5 elements.

**2. Size is part of the type.** You cannot pass a `[5]int` to a function that expects a `[3]int`. They are different types. This makes arrays inflexible for function parameters.

These two limitations are why **slices exist**. Slices provide a dynamic, flexible wrapper around arrays. In real Go code, you will use slices 99% of the time.

## Multi-Dimensional Arrays

You can create arrays of arrays for multi-dimensional data:

```go
var grid [3][3]int // 3x3 grid of integers

grid[0][0] = 1
grid[1][1] = 2
grid[2][2] = 3

// Initialize with literal
board := [2][2]string{
    {"X", "O"},
    {"O", "X"},
}

fmt.Println(board[0][1]) // "O"
```

In practice, multi-dimensional slices are more common than multi-dimensional arrays, for the same reason: flexibility.

## Iterating Over Arrays

You can use a classic `for` loop or `range`:

```go
fruits := [3]string{"apple", "banana", "cherry"}

// Using range
for index, value := range fruits {
    fmt.Println(index, value)
}

// Only value
for _, value := range fruits {
    fmt.Println(value)
}

// Only index
for index := range fruits {
    fmt.Println(index, fruits[index])
}
```

## A Personal Note

I almost never use arrays in real Go code. Every time I think I need an array, I end up using a slice instead. Slices are just more flexible: you can grow them, shrink them, pass them to functions without worrying about size, and they work better with Go's standard library.

But understanding arrays matters because slices are built on top of arrays. When you create a slice, there is an array underneath. When you understand how arrays work (value semantics, fixed size, contiguous memory), you understand why slices behave the way they do.

Think of arrays as the skeleton and slices as the body. The skeleton is hidden inside, but it gives the body its structure.
