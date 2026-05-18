# Slices in Go

## What Is a Slice?

A **slice** is a dynamic, flexible view into an array. Unlike arrays, slices can grow and shrink. They do not have a fixed size. This makes them the go-to data structure for lists and collections in Go.

If I had to pick one data structure I use most in Go, it would be slices. They are everywhere. Function parameters, return values, loop iterations, JSON arrays. Slices are the workhorse of Go.

## Creating Slices

There are several ways to create a slice:

**1. Slice literal:**

```go
nums := []int{1, 2, 3, 4, 5}
names := []string{"Alice", "Bob", "Charlie"}
```

Notice there is no size in the brackets. `[]int` is a slice type, `[5]int` is an array type.

**2. Using make():**

```go
s1 := make([]int, 5)     // length 5, capacity 5, all zeros
s2 := make([]int, 3, 10) // length 3, capacity 10, all zeros
```

`make()` creates a slice with a specified length and optional capacity. The elements are initialized to the zero value of the type.

**3. Slicing an existing array:**

```go
arr := [5]int{10, 20, 30, 40, 50}
s := arr[1:4] // [20, 30, 40]
```

This creates a slice that is a "view" into the array `arr`. It shares the same underlying memory.

**4. Empty slice:**

```go
s := []int{} // empty but initialized
```

## Slice Anatomy: Pointer, Length, Capacity

A slice has three components under the hood:

- **Pointer**: points to the start of the underlying array
- **Length** (`len`): number of elements currently in the slice
- **Capacity** (`cap`): total number of elements in the underlying array from the pointer to the end

```go
s := make([]int, 3, 10)
fmt.Println(len(s)) // 3 - has 3 elements
fmt.Println(cap(s)) // 10 - can hold up to 10 before reallocating
```

You can think of it like this: the length is how much of the apartment you are using, and the capacity is the total size of the apartment. You can expand into unused space without moving.

## append()

The `append()` function adds elements to a slice. It returns a **new slice** (which might or might not share the same underlying array):

```go
s := []int{1, 2, 3}
s = append(s, 4)       // [1, 2, 3, 4]
s = append(s, 5, 6, 7) // [1, 2, 3, 4, 5, 6, 7]
```

**Important:** You must reassign the result of `append()` back to the slice variable. If you do not, you lose the updated slice:

```go
s := []int{1, 2, 3}
append(s, 4)   // WRONG: result is discarded
fmt.Println(s) // [1, 2, 3] - unchanged!

s = append(s, 4) // CORRECT: reassign the result
fmt.Println(s)   // [1, 2, 3, 4]
```

When a slice runs out of capacity, `append()` creates a new, larger underlying array and copies everything over. The new array is usually double the old capacity:

```go
s := make([]int, 3, 3) // len=3, cap=3
s = append(s, 4)       // new array allocated, cap might be 6 or 8
```

## Slice Expressions

There are two kinds of slice expressions:

**Simple slice expression:**

```go
arr := [5]int{10, 20, 30, 40, 50}
s := arr[1:4] // elements at index 1, 2, 3 -> [20, 30, 40]
```

The slice includes elements from index `low` up to (but not including) index `high`.

**Full slice expression (three-index):**

```go
arr := [5]int{10, 20, 30, 40, 50}
s := arr[1:3:4] // low:high:max
// Elements: [20, 30]
// Length: 3 - 1 = 2
// Capacity: 4 - 1 = 3
```

The third index `max` controls the capacity of the resulting slice. This is useful when you want to prevent `append()` from overwriting elements in the original array beyond your slice:

```go
s1 := arr[1:3]    // len=2, cap=4
s2 := arr[1:3:3]  // len=2, cap=2

s1 = append(s1, 99) // overwrites arr[3]!
s2 = append(s2, 99) // safe: new array allocated
```

The full slice expression protects you from accidental sharing. I use it when slicing arrays that I want to keep intact.

## nil Slice vs Empty Slice

A **nil slice** has no underlying array:

```go
var s []int
fmt.Println(s)     // []
fmt.Println(len(s)) // 0
fmt.Println(cap(s)) // 0
fmt.Println(s == nil) // true
```

An **empty slice** has an underlying array (of length 0):

```go
s := []int{}
fmt.Println(s)     // []
fmt.Println(len(s)) // 0
fmt.Println(cap(s)) // 0
fmt.Println(s == nil) // false
```

Both have zero length and work the same with `append()` and `range`. The difference is whether `s == nil` is true. In practice, a nil slice is often preferred for "no data yet" and an empty slice for "we checked and there is nothing."

## Copying Slices with copy()

The `copy()` function copies elements from one slice to another:

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)

n := copy(dst, src)
fmt.Println(dst) // [1, 2, 3]
fmt.Println(n)   // 3 (number of elements copied)
```

`copy()` copies the minimum of `len(dst)` and `len(src)` elements. It does not grow the destination slice.

This is the proper way to create an independent copy of a slice:

```go
original := []int{1, 2, 3, 4, 5}
clone := make([]int, len(original))
copy(clone, original)
clone[0] = 99
fmt.Println(original) // [1, 2, 3, 4, 5] - unchanged
```

## 2D Slices

Go does not have built-in 2D slices, but you can create them as slices of slices:

```go
rows := 3
cols := 4
grid := make([][]int, rows)
for i := range grid {
    grid[i] = make([]int, cols)
}

grid[0][1] = 5
grid[2][3] = 9
fmt.Println(grid)
// [[0 5 0 0] [0 0 0 0] [0 0 0 9]]
```

Each inner slice is independent, so they can have different lengths if needed (jagged arrays).

## Common Mistakes

**1. Forgetting to reassign after append():**

```go
s := []int{1, 2, 3}
append(s, 4)  // WRONG: result discarded
s = append(s, 4) // CORRECT
```

**2. Shared backing array surprises:**

```go
arr := [5]int{10, 20, 30, 40, 50}
s1 := arr[1:4] // [20, 30, 40]
s1[0] = 99
fmt.Println(arr) // [10, 99, 30, 40, 50] - arr was modified!
```

Slices share the underlying array. Modifying a slice element modifies the original array. To avoid this, use `copy()`.

**3. Appending to a slice that shares an array:**

```go
arr := [5]int{10, 20, 30, 40, 50}
s := arr[1:3]   // [20, 30], cap=4
s = append(s, 99) // [20, 30, 99]
fmt.Println(arr)  // [10, 20, 30, 99, 50] - arr[3] was overwritten!
```

Use the three-index slice expression to prevent this: `arr[1:3:3]`.

## A Personal Note

Slices are the most used data structure in Go. I use them for everything: lists of items, buffers, results from databases, JSON arrays, and more. Understanding how they work under the hood (pointer, length, capacity, shared arrays) was a turning point in my Go learning.

The shared backing array thing caught me off guard more than once. I would modify a slice and wonder why my original data changed. Now I always think: "does this slice share memory with anything?" If yes, I either copy it or use the three-index slice expression.

Once you internalize the slice internals, everything makes sense. The `append()` behavior, the capacity growth, the sharing. It all follows logically from the simple structure of pointer + length + capacity.
