# Chapter 5: Computer Science Fundamentals

## Why Bother With CS Fundamentals?

I will be honest with you. When I first started learning Go, I skipped all the "theory" stuff. I just wanted to write code, build APIs, and ship things. Who cares about how the CPU works? Who cares about operating systems? I just wanted my goroutines to run and my channels to work.

But then something funny happened. I kept running into problems I could not explain.

Why does my Go server slow down when I spawn 10,000 goroutines? Why does my channel sometimes deadlock? Why is my concurrent program not actually faster on a single core? Why does the OS matter for my Go code?

I realized I was trying to use powerful tools without understanding the machine underneath them. It is like trying to drive a race car without knowing how the engine works. You can go fast, but when something breaks, you have no idea why.

**You cannot truly understand goroutines without understanding threads.**

**You cannot understand channels without understanding concurrency.**

**You cannot understand why Go is fast without understanding context switching.**

These are not just academic facts. They are the foundation that makes Go's concurrency model make sense. Once I learned these concepts, everything clicked into place.

## This Is Not Academic Stuff

I know what you are thinking. "CS fundamentals" sounds like a university course with boring textbooks and exams. That is not what this chapter is.

This chapter is about practical knowledge that makes you a better Go programmer. Every concept here connects directly to something you do in Go:

- **Computer architecture** helps you write cache-friendly code that runs faster
- **Operating system basics** help you understand why file I/O and networking behave the way they do
- **Processes and context switching** help you understand why goroutines are more efficient than OS threads
- **Concurrency vs parallelism** helps you design better programs that actually benefit from multiple cores
- **Threads and stacks** help you understand why Go can run millions of goroutines but not millions of OS threads

These concepts are not theory for the sake of theory. They are the "why" behind the "how." When you understand why goroutines are cheap, you know when to use them. When you understand why context switching is expensive, you know how to avoid it. When you understand the difference between concurrency and parallelism, you design better programs.

## How Learning CS Concepts Changed My Go

Before I learned these fundamentals, my approach to Go was mostly copy-paste from Stack Overflow. "Oh, I need concurrency? Add a goroutine. Oh, I need synchronization? Add a mutex." It worked, sort of, but I did not really know why.

After learning the basics in this chapter, I started making better decisions:

- I understood when to use goroutines and when not to
- I understood why channels work the way they do
- I understood why my program was slow and how to fix it
- I could reason about concurrency instead of just guessing
- I could debug race conditions instead of just adding more locks
- I understood why my "parallel" program was not actually running in parallel

It was like someone turned on the lights in a dark room. The code was the same, but I could finally see what was going on.

A specific example: I once wrote a Go service that created a new goroutine for every incoming request. It worked fine in development with 10 requests per second. But in production, with 10,000 requests per second, it fell apart. If I had understood how the Go scheduler works, how goroutines are mapped to OS threads, and how context switching works, I would have designed the service differently from the start.

## The Layered Model

One way to think about it is that your Go code sits on top of several layers, and each layer matters:

```
+-----------------------------------+
|       Your Go Application         |   <- You write code here
+-----------------------------------+
|       Go Runtime                  |   <- Goroutines, scheduler, GC
+-----------------------------------+
|       Operating System            |   <- Processes, threads, syscalls
+-----------------------------------+
|       Hardware                    |   <- CPU, RAM, disk, network
+-----------------------------------+
```

When something goes wrong in your Go program, the cause could be in any of these layers. A slow program might be caused by bad Go code, by the Go runtime making unexpected scheduling decisions, by the OS doing too many context switches, or by the hardware not having enough cache. Understanding the whole stack helps you find and fix problems faster.

## What We Will Cover

In this chapter, we will go through the CS fundamentals that matter most for Go developers:

1. **Computer Architecture** - How the hardware works and why it matters for your code
2. **Operating System Basics** - What the OS does for you and why you should care
3. **CPU and Processes** - How programs actually run on the CPU
4. **Context Switching** - The hidden cost of multitasking
5. **Process Control Block** - How the OS keeps track of processes
6. **Concurrency** - Dealing with multiple things at once
7. **Parallelism** - Doing multiple things at the same time
8. **Threads** - The building blocks of concurrent programs
9. **Separate Stacks for Threads** - Why goroutines are so lightweight

Each section will have code examples in Go and diagrams to help you visualize what is happening. I am writing this as someone who is learning these concepts, not as an expert lecturing you. If I can understand this, you can too.

## A Note on How to Read This Chapter

You do not need to read this chapter all at once. Each section builds on the previous ones, but they are also self-contained enough that you can jump to the one that interests you most.

If you are completely new to CS fundamentals, I recommend reading them in order. The concepts build on each other: you need to understand processes before context switching, and you need to understand threads before goroutine stacks.

If you already know some of these concepts, feel free to skip around. Just know that each section references earlier ones, so you might need to go back occasionally.

Let us start from the bottom, with the hardware itself.
