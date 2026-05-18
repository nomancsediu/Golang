# If, Else, and Switch in Go

## Making Decisions in Your Code

Programs are not just a straight line of instructions. They need to make decisions. "Is the user logged in?" "Is the number positive?" "Which day of the week is it?" These are all decisions your code needs to handle. In Go, we use **if**, **else**, and **switch** to control the flow of our program. Let me show you how each one works.

## The If Statement

The basic `if` statement in Go looks like this:

```go
package main

import "fmt"

func main() {
    age := 20

    if age >= 18 {
        fmt.Println("You are an adult")
    }
}
```

A few things to notice right away:

- **No parentheses** around the condition. `if age >= 18` not `if (age >= 18)`. You can add them if you really want to, but Go style says leave them out.
- **Braces are mandatory.** You cannot write `if x > 0 fmt.Println("positive")` on one line. The `{ }` must always be there.
- **The opening brace must be on the same line as the if.** This is not just style, it is a rule. Go enforces this.

## If-Else

When you want to handle both the true and false cases:

```go
score := 75

if score >= 60 {
    fmt.Println("You passed!")
} else {
    fmt.Println("You failed. Try again.")
}
```

Straightforward. If the condition is true, run the first block. Otherwise, run the else block.

## If-Else If-Else

For multiple conditions:

```go
score := 85

if score >= 90 {
    fmt.Println("Grade: A")
} else if score >= 80 {
    fmt.Println("Grade: B")
} else if score >= 70 {
    fmt.Println("Grade: C")
} else if score >= 60 {
    fmt.Println("Grade: D")
} else {
    fmt.Println("Grade: F")
}
```

Go evaluates conditions from top to bottom and runs the first one that is true. If none match, the else block runs.

## If with Init Statement

This is one of my favorite features in Go. You can declare a variable right inside the `if` statement. The variable only exists within the if-else blocks:

```go
package main

import "fmt"

func main() {
    if score := 85; score >= 90 {
        fmt.Println("Grade: A")
    } else if score >= 80 {
        fmt.Println("Grade: B")
    } else if score >= 70 {
        fmt.Println("Grade: C")
    } else {
        fmt.Println("Grade: F")
    }

    // score is not accessible here
    // fmt.Println(score) // ERROR: undefined: score
}
```

The `score := 85;` part is the **init statement**. It runs before the condition is checked. The variable `score` is scoped to the entire if-else chain. After the closing brace, it disappears.

This is really useful when you want to compute a value just for the purpose of a condition check. I use this pattern a lot, especially with error handling:

```go
if err := doSomething(); err != nil {
    fmt.Println("Error:", err)
}
```

## The Switch Statement

When you have many conditions to check against a single value, `switch` is cleaner than a long chain of if-else:

```go
day := "Monday"

switch day {
case "Monday":
    fmt.Println("Start of the work week")
case "Friday":
    fmt.Println("Almost weekend!")
case "Saturday", "Sunday":
    fmt.Println("It is the weekend!")
default:
    fmt.Println("Just a regular day")
}
```

### Key Differences from Other Languages

If you are coming from C, Java, or JavaScript, pay attention:

- **No `break` needed.** Go automatically breaks after each case. You do not have to write `break` at the end of every case. This is the default behavior.
- **No fallthrough by default.** In C and Java, execution falls through to the next case unless you break. In Go, it does not. Each case is independent.

```go
x := 2

switch x {
case 1:
    fmt.Println("One")
    // no break needed, Go does it automatically
case 2:
    fmt.Println("Two")
case 3:
    fmt.Println("Three")
}
// Output: Two (just "Two", not "Two" + "Three")
```

### Using Fallthrough Explicitly

If you actually want fallthrough behavior, you must use the `fallthrough` keyword explicitly:

```go
x := 2

switch x {
case 1:
    fmt.Println("One")
    fallthrough
case 2:
    fmt.Println("Two")
    fallthrough
case 3:
    fmt.Println("Three")
}
// Output:
// Two
// Three
```

I rarely use `fallthrough`, but it is good to know it exists.

### Multiple Values in a Case

You can match multiple values in a single case by separating them with commas:

```go
character := 'a'

switch character {
case 'a', 'e', 'i', 'o', 'u':
    fmt.Println("Vowel")
default:
    fmt.Println("Consonant")
}
```

This is much cleaner than writing five separate cases or a complex if condition.

### Switch Without a Condition

This is a cool trick. You can write a switch with no variable after it, and each case is an independent condition:

```go
score := 78

switch {
case score >= 90:
    fmt.Println("Grade: A")
case score >= 80:
    fmt.Println("Grade: B")
case score >= 70:
    fmt.Println("Grade: C")
case score >= 60:
    fmt.Println("Grade: D")
default:
    fmt.Println("Grade: F")
}
// Output: Grade: C
```

This is basically a cleaner way to write if-else if-else chains. I prefer this over long if-else chains because it is easier to read and each case is visually distinct.

### Switch with Init Statement

Just like if, switch can have an init statement:

```go
switch num := 15; {
case num%2 == 0:
    fmt.Println("Even")
case num%2 != 0:
    fmt.Println("Odd")
}
// Output: Odd
```

The variable `num` is declared and then used in the switch. It only exists within the switch block.

## Type Switch

Go also has a special kind of switch for checking types, which is useful with interfaces. I have not learned interfaces deeply yet, but here is a quick preview:

```go
func checkType(x interface{}) {
    switch v := x.(type) {
    case int:
        fmt.Println("Integer:", v)
    case string:
        fmt.Println("String:", v)
    case bool:
        fmt.Println("Boolean:", v)
    default:
        fmt.Println("Unknown type")
    }
}
```

This will make more sense when we cover interfaces later. For now, just know it exists.

## If vs Switch: When to Use Which

Here is my simple guideline:

- Use **if** when you have one or two simple conditions
- Use **if-else if** when conditions are complex and involve different variables
- Use **switch** when you are comparing one value against many possible matches
- Use **switch without condition** when you have multiple independent conditions and want cleaner code than if-else if

## Putting It All Together

Here is a more complete example that uses if, else, and switch together:

```go
package main

import "fmt"

func main() {
    temperature := 25

    // Using if-else
    if temperature > 35 {
        fmt.Println("It is really hot outside!")
    } else if temperature > 25 {
        fmt.Println("It is warm and nice")
    } else if temperature > 15 {
        fmt.Println("It is comfortable")
    } else {
        fmt.Println("It is cold, wear a jacket")
    }

    // Using switch for the same logic
    switch {
    case temperature > 35:
        fmt.Println("It is really hot outside!")
    case temperature > 25:
        fmt.Println("It is warm and nice")
    case temperature > 15:
        fmt.Println("It is comfortable")
    default:
        fmt.Println("It is cold, wear a jacket")
    }

    // Switch with a value
    season := "summer"
    switch season {
    case "spring":
        fmt.Println("Flowers are blooming")
    case "summer":
        fmt.Println("Time for the beach")
    case "autumn":
        fmt.Println("Leaves are falling")
    case "winter":
        fmt.Println("Stay warm!")
    default:
        fmt.Println("Unknown season")
    }
}
```

Both the if-else chain and the switch produce the same result. The switch version is just easier to read when conditions get longer.

Now that we can make decisions in our code, let us learn about **functions**, which let us organize our code into reusable blocks.
