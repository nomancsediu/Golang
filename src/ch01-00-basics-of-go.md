# Chapter 1: Basics of Go

## Why I Chose to Learn Go

I remember the first time I heard about Go. Someone said "it compiles fast and has concurrency built in" and I thought, okay, every language claims something cool. But then I actually tried it, and honestly, it felt different. Not in a hype way, but in a "wow this actually makes sense" way.

Go, also called Golang, was created at Google by three people: **Rob Pike**, **Ken Thompson**, and **Robert Griesemer**. If you do not know who Ken Thompson is, he literally co-created Unix. These are not random people. They built Go because they were tired of the complexity of C++ and Java at Google. They wanted something simple, fast, and practical.

And that is exactly what Go is.

## What Makes Go Special

Here are the things that made me say "yes, this is the language I want to learn":

**Fast Compilation** - Go compiles incredibly fast. Like, you blink and it is done. Coming from languages where you wait minutes for a build, this feels magical. The compiler is fast because the language is simple by design. Large Go projects compile in seconds, not minutes. This completely changes the development experience. You spend more time coding and less time waiting.

**Built-in Concurrency** - This is the big one. Go has **goroutines** and **channels** built right into the language. You do not need external libraries or complex frameworks to write concurrent code. You just type `go doSomething()` and boom, it runs concurrently. I have not fully mastered this yet, but even the basics are powerful. Goroutines are lightweight threads managed by the Go runtime, and they use very little memory. You can run thousands of them without breaking a sweat.

**Simple Syntax** - Go has only **25 keywords**. Yes, you read that right. Twenty-five. Compare that to C++ which has over 80, or Java which has around 50. This means the language is small enough to fit in your head. You can actually remember all of it. The full list of keywords is: break, case, chan, const, continue, default, defer, else, fallthrough, for, func, go, goto, if, import, interface, map, package, range, return, select, struct, switch, type, var. That is it. That is the whole language.

**Great for Backend Development** - Go was designed for building servers and services. It has an excellent standard library for HTTP, JSON, file I/O, and networking. Many famous tools are written in Go: Docker, Kubernetes, Terraform, Prometheus, and many more. When you learn Go, you are learning the language that powers modern cloud infrastructure.

**Static Binary Output** - When you build a Go program, you get a single binary file. No dependencies, no runtime needed, no "it works on my machine" problems. You just copy the binary and run it. This is huge for deployment. No need to install a runtime on the server. No dependency conflicts. Just build once and deploy everywhere.

**Cross-Platform Compilation** - Go can compile for different operating systems and architectures from a single machine. Want to build a Linux binary from your Mac? Just set an environment variable and run `go build`. This is incredibly convenient for targeting multiple platforms.

## A Personal Note

I am not an expert. I am learning Go step by step, and I am writing this as I go. Some things confuse me, some things click right away. But I can already tell that Go rewards patience. The simplicity is not a limitation. It is a feature.

When I first saw that Go does not have classes or inheritance, I was confused. "How do I organize my code?" I wondered. But then I learned about **structs** and **interfaces** and it started making sense. Go takes a different approach, and once you understand the philosophy behind it, everything falls into place.

Go also enforces a specific code style. There is a tool called `gofmt` that automatically formats your code. No more debates about tabs vs spaces or where to put braces. Everyone writes Go the same way, and that makes reading other people's code much easier. I love this.

Another thing that surprised me: Go compiles so fast that you can use it like a scripting language. There is even a tool called `go run` that compiles and runs your code in one step. It feels like Python, but with the safety of a compiled language.

## What This Section Covers

This section is all about the **basics**. We will cover:

- **Hello World** - Your first Go program and how to run it
- **Variables and Data Types** - How to store and work with data
- **If, Else, and Switch** - Controlling the flow of your program
- **Functions** - The building blocks of any Go program
- **Function Return Values** - One of Go's coolest features: multiple return values
- **More Function Examples** - Practice with real, useful functions
- **Why Functions Are Needed** - Understanding the "why" behind functions

Each chapter has code examples that I wrote and tested myself. I will explain what each part does and share my own "aha moments" along the way.

## Quick Facts About Go

Before we dive in, here are some quick facts to keep in mind:

- **Created**: 2007, released publicly in 2009
- **Created by**: Rob Pike, Ken Thompson, Robert Griesemer at Google
- **Keywords**: Only 25 (for, if, else, switch, func, return, var, const, etc.)
- **Typing**: Statically typed (types are checked at compile time)
- **Compilation**: Compiles to native machine code
- **Garbage Collection**: Yes, automatic memory management
- **Concurrency**: Goroutines and channels built in
- **Open Source**: Yes, hosted on GitHub at github.com/golang/go
- **Mascot**: The Go gopher, a cute blue character designed by Renee French

## Who Uses Go?

Go is not just a niche language. Some of the biggest companies in the world rely on it:

- **Google** uses Go internally for many services and tools
- **Docker** the entire container platform is written in Go
- **Kubernetes** the container orchestration system that runs the cloud
- **Terraform** infrastructure as code tool by HashiCorp
- **Prometheus** monitoring and alerting system
- **Uber** uses Go for many of their backend microservices
- **Twitch** uses Go for their chat system handling millions of users
- **SoundCloud** was an early adopter of Go for backend services

When I saw this list, it gave me confidence. Learning Go is not a gamble. It is a solid investment in a language that the industry has already validated.

## My Learning Approach

Before we start, I want to share how I am approaching this learning journey:

1. **Type every example by hand.** Copy-pasting teaches you nothing. Typing builds muscle memory.
2. **Experiment with the code.** After each example, I change something and see what happens. What if I pass a float to an int function? What if I remove a return statement?
3. **Read the error messages.** Go's compiler errors are incredibly helpful. They tell you exactly what is wrong and where. I have learned more from error messages than from any tutorial.
4. **Build small projects.** After learning the basics, I try to build something tiny. A calculator, a todo list, a number guessing game. It does not have to be impressive. It just has to work.
5. **Do not rush.** Understanding the basics well is more important than rushing to advanced topics. A shaky foundation will crumble later.

If you are coming from Python or JavaScript, Go will feel strict at first. No implicit type conversions, no unused variables allowed, and you must handle errors explicitly. But trust me, that strictness becomes your friend. It catches bugs before they catch you.

Let us start with the classic: Hello World.
