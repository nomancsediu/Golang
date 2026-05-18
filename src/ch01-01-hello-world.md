# Hello World in Go

## The Moment Every Programmer Loves

Every programming journey starts the same way: Hello World. It is a tradition, a ritual, and honestly, it is exciting. Seeing those two words appear on your screen means your setup works, your tools are ready, and you are officially writing code in a new language. Let me walk you through how I did it.

## Installing Go

Before writing any code, you need to install Go on your computer. Here is how:

1. Go to **golang.org/dl** (the official Go website)
2. Download the installer for your operating system (Windows, macOS, or Linux)
3. Run the installer and follow the steps
4. Open your terminal and type:

```bash
go version
```

If you see something like `go version go1.22.0 linux/amd64`, you are good to go. If you get an error like "command not found", you might need to restart your terminal or add Go to your PATH manually.

On Linux, you can also install Go using the package manager, but downloading from the official site gives you the latest version.

## Setting Up Your Workspace

Go used to require a specific workspace structure with a `GOPATH` folder. That is mostly optional now thanks to **Go modules**. Here is what I did:

```bash
mkdir my-first-go-project
cd my-first-go-project
go mod init github.com/yourname/my-first-go-project
```

The `go mod init` command creates a `go.mod` file. This file tracks your project's dependencies. Think of it like `package.json` in Node.js or `requirements.txt` in Python. The name you give it can be anything, but the convention is to use a path-like format.

## Your First Go Program

Now the fun part. Create a file called `main.go` and type this:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

That is it. Five lines. Your first Go program. Let me break down every single piece.

### What Each Part Means

**`package main`** - Every Go file starts with a package declaration. The `main` package is special. It tells Go "this is an executable program, not a library." If you name it anything other than `main`, Go will compile it as a library package that other code can import.

**`import "fmt"`** - This imports the `fmt` package from Go's standard library. The `fmt` package handles formatted input and output. You will use it a lot. `fmt` stands for "format."

**`func main()`** - This defines the `main` function. In Go, the `main` function in the `main` package is the entry point of your program. When you run your program, Go starts executing from here. No arguments, no return value. Just `func main()`.

**`fmt.Println("Hello, World!")`** - This calls the `Println` function from the `fmt` package. It prints the text to the terminal and adds a newline at the end. The `Println` part is a capitalized name, which in Go means it is an **exported** function (public, accessible from other packages).

## Running Your Program

There are two main ways to run a Go program:

### Using `go run`

```bash
go run main.go
```

This compiles and runs your code in one step. It is great for development and quick testing. The compiled binary is temporary and gets deleted after the program finishes.

### Using `go build`

```bash
go build main.go
./main
```

This compiles your code into a permanent executable binary file. On Linux and Mac, you run it with `./main`. On Windows, it creates `main.exe` and you run it by typing `main.exe`.

The `go build` command is what you use when you want to deploy your program. The resulting binary is self-contained. No Go installation needed on the target machine.

## Trying More with fmt

The `fmt` package is your best friend in Go. Here are a few more things it can do:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
    fmt.Println("My name is Noman")
    fmt.Println("I am learning Go!")

    // Print without a newline
    fmt.Print("Same ")
    fmt.Print("line\n")

    // Print with formatting
    name := "Go"
    year := 2009
    fmt.Printf("%s was released in %d\n", name, year)
}
```

**`fmt.Print()`** - Prints without adding a newline at the end
**`fmt.Println()`** - Prints and adds a newline at the end
**`fmt.Printf()`** - Prints with format verbs like `%s` for strings and `%d` for integers

The output would be:

```
Hello, World!
My name is Noman
I am learning Go!
Same line
Go was released in 2009
```

## A Few Things I Found Interesting

When I wrote my first Go program, a few things surprised me in a good way:

- **Compilation is instant.** I literally could not measure the time it took. Coming from larger projects in other languages, this felt amazing.
- **No semicolons needed.** Go adds them automatically during compilation. You just write clean code without worrying about `;` at the end of every line.
- **Unused imports cause errors.** If you import a package and do not use it, Go refuses to compile. This keeps your code clean. At first it annoyed me, but now I appreciate it.
- **Unused variables also cause errors.** Same philosophy. Go does not let you be sloppy.

These might feel restrictive at first, but they actually help you write better code. Go is opinionated, and that opinion is: keep things clean and simple.

## What is Next?

Now that you can write and run a Go program, it is time to learn about **variables and data types**. That is where you start storing information and making your programs actually do something useful. Let us keep going.
