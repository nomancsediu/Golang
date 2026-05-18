# Ch03 00 Advanced Functions

Now that you know the basics of functions, how to declare them, how to return values, and how scope works, it is time to step up. This section is about the powerful stuff. The stuff that makes Go functions truly special and the stuff you will see in every real Go codebase.

## Functions Are First-Class Citizens

In Go, **functions are first-class citizens**. What does that mean? It means functions are treated like any other value. You can assign a function to a variable. You can pass a function as an argument to another function. You can return a function from a function. You can store functions in data structures like slices and maps.

When I first heard "first-class citizen," it sounded like some academic term that would not matter in practice. I was wrong. This concept is everywhere in real Go code. Every time you use `http.HandleFunc`, you are passing a function. Every time you write `go func()`, you are using a function as a value. Every time you write middleware, you are returning a function from a function.

If you come from a language like C or Java, this might feel new. In C, you have function pointers, but they are clunky. In Java, you have lambdas, but they are tied to interfaces. In Go, functions as values feel natural. The syntax is clean and the behavior is predictable.

## A Quick Example

Before we dive into the chapters, let me show you a quick taste of what first-class functions look like in Go:

```go
package main

import "fmt"

// A function that takes another function as an argument
func apply(nums []int, transform func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = transform(n)
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    // Passing an anonymous function as the transform
    doubled := apply(numbers, func(n int) int {
        return n * 2
    })

    fmt.Println(doubled) // [2 4 6 8 10]
}
```

In this example, `apply` is a higher-order function. It takes a slice and a function, and it applies that function to every element. The function we pass in is an anonymous function that doubles each number. This is the kind of pattern you will see everywhere once you start reading Go code.

If this looks confusing right now, do not worry. By the end of this section, this pattern will feel second nature.

## What This Section Covers

This section goes deep into the advanced side of Go functions. Here is what we will learn, chapter by chapter:

**Function Types** - How to define your own function types with the `type` keyword and why they matter. Function types are the foundation for understanding everything else in this section. They let you describe the shape of a function the same way `int` describes the shape of a number.

**Named Functions** - The functions you already know, but with more detail. We will cover exported vs unexported functions, naming conventions, and how functions form the API of a package. Good naming is not just style. It is how you communicate with other developers.

**Anonymous Functions** - Functions without names. These show up all the time in Go, especially with goroutines and callbacks. You will learn how to define them, when to use them, and when to avoid them.

**IIFE** - Immediately Invoked Function Expressions. A fancy name for a simple idea: a function that runs right where you define it. Great for keeping scope clean and for goroutine patterns.

**Function Expressions** - Assigning a function to a variable. This is where functions start behaving like data. You will see how function expressions differ from named functions and how they capture variables from their surroundings.

**Parameters vs Arguments** - A common source of confusion. Parameters are what the function expects. Arguments are what you actually pass. We will also cover variadic parameters, pointer parameters, and named return values.

**First Order Functions** - Simple functions that take data in and return data out. No function arguments, no function returns. Just plain and predictable. Most of the functions you write will be first-order.

**Higher Order Functions** - Functions that take other functions as arguments or return functions. This is where the real power lives. Filter, map, middleware, and strategy patterns all rely on higher-order functions. This chapter will change how you think about code structure.

**Closures** - Functions that capture variables from their outer scope. Closures are one of the most important concepts in Go. They are also the source of one of the most common bugs in Go programs, the goroutine loop variable bug. We will cover that bug, how to fix it, and how Go 1.22+ changes the game.

## Why This Section Matters

I used to think advanced function concepts were just academic. Then I started reading real Go codebases and realized that almost every interesting piece of Go code uses these concepts. Middleware in web frameworks uses higher-order functions. Goroutines use anonymous functions and closures. Event systems use function types. Dependency injection uses function types and closures. The standard library itself is full of these patterns.

If you skip this section, you will still be able to write Go code. But you will struggle to read other people's Go code, and you will miss out on some of the most elegant patterns the language offers. These concepts are not optional extras. They are core to writing idiomatic Go.

I remember the first time I read the source code for Go's `net/http` package. It was full of function types, closures, and higher-order functions. I could not follow it at all. After learning these concepts, I went back and read it again. It made sense. Not easy, but sensible. That is the difference these concepts make.

## How to Approach This Section

Some of these concepts, especially closures, can feel confusing at first. That is normal. Here is my advice:

**Read the code examples carefully.** Do not just skim them. Look at what each line is doing.

**Type out the examples yourself.** Copying code from a page does not teach you as much as writing it. Make typos. Fix them. That is how you learn.

**Modify the examples.** Change the numbers. Change the function signatures. Break things on purpose and see what error messages you get.

**Take breaks.** If closures are not clicking, move on to the next chapter and come back later. Sometimes your brain needs time to process.

**Connect the dots.** Each chapter builds on the previous ones. Function types lead to anonymous functions. Anonymous functions lead to closures. Closures lead to middleware. It all connects.

Let us start with function types in the next chapter.
