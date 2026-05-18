# Channels in Go

## What is a Channel?

A **channel** is a pipe that goroutines use to communicate with each other. Think of it like a real pipe or a conveyor belt. One goroutine puts something in, and another goroutine takes it out. That is it. Simple concept, but incredibly powerful.

Channels are how Go implements its core concurrency philosophy: **"Do not communicate by sharing memory; share memory by communicating."** This took me a while to understand. In other languages, you share a variable between threads and protect it with locks. In Go, you send data through a channel, and only one goroutine owns the data at any time. No locks needed.

## Creating a Channel

You create a channel using the `make` function:

```go
ch := make(chan int)       // Unbuffered channel of integers
ch := make(chan string)    // Unbuffered channel of strings
ch := make(chan bool, 3)   // Buffered channel of booleans with capacity 3
```

The type after `chan` specifies what kind of data the channel can carry. A `chan int` can only carry integers. A `chan string` can only carry strings.

## Sending and Receiving

The arrow operator `<-` is used to send and receive:

```go
ch <- value    // Send value to channel
value := <-ch  // Receive from channel and assign to value
<-ch           // Receive and discard the value
```

The arrow always points in the direction of data flow. `ch <- value` means "value flows into ch." `<-ch` means "data flows out of ch."

Here is a complete example:

```go
package main

import "fmt"

func main() {
    ch := make(chan string)

    go func() {
        ch <- "Hello from goroutine!"
    }()

    msg := <-ch
    fmt.Println(msg)
}
```

The goroutine sends a message, and the main goroutine receives it. They synchronize through the channel.

## Blocking Behavior

This is the most important thing to understand about channels: **unbuffered channels block.**

- **Sending blocks** until another goroutine is ready to receive
- **Receiving blocks** until another goroutine sends

This means that an unbuffered channel is not just a data pipe. It is also a **synchronization point**. When a goroutine sends on a channel, it waits until someone is there to receive. When a goroutine receives, it waits until someone sends. They meet at the channel.

```go
package main

import "fmt"

func main() {
    ch := make(chan int)

    go func() {
        fmt.Println("Goroutine: about to send")
        ch <- 42
        fmt.Println("Goroutine: sent!") // This prints AFTER main receives
    }()

    fmt.Println("Main: about to receive")
    val := <-ch
    fmt.Println("Main: received", val)
}
```

The output will always show the sending and receiving happening in a coordinated way. Neither side moves past the channel operation until the other side is ready.

## Buffered Channels

A **buffered channel** has a capacity. It can hold that many values before blocking:

```go
ch := make(chan int, 3) // Buffer size of 3

ch <- 1 // Does not block (buffer has space)
ch <- 2 // Does not block (buffer has space)
ch <- 3 // Does not block (buffer is now full)
// ch <- 4 // Would block! Buffer is full, must wait for a receive

val := <-ch // Receives 1, buffer now has space for one more
```

Buffered channels are useful when the sender and receiver operate at different speeds. The buffer acts as a cushion so they do not have to be perfectly synchronized.

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 5) // Buffered channel

    // Sender can send multiple values without blocking
    for i := 0; i < 5; i++ {
        ch <- i
        fmt.Printf("Sent %d\n", i)
    }

    // Close the channel after sending is done
    close(ch)

    // Receiver reads all values
    for val := range ch {
        fmt.Printf("Received %d\n", val)
    }
}
```

## Closing Channels

You can close a channel to signal that no more values will be sent:

```go
close(ch)
```

After closing:

- Sending on a closed channel causes a **panic**
- Receiving from a closed channel returns the **zero value** of the channel's type
- You can check if a channel is closed using the two-value receive:

```go
value, ok := <-ch
if !ok {
    fmt.Println("Channel is closed!")
}
```

### Iterating with Range

The most common way to read all values from a channel is using `range`:

```go
ch := make(chan int, 3)
ch <- 10
ch <- 20
ch <- 30
close(ch) // Must close before ranging!

for val := range ch {
    fmt.Println(val) // Prints 10, 20, 30
}
```

**Important**: You must close the channel before using `range`, otherwise the range loop will block forever waiting for more values.

## Channel Direction

You can restrict a channel to be **send-only** or **receive-only** in function signatures. This is a compile-time safety feature:

```go
func sender(ch chan<- int) {
    // Can only send to ch
    ch <- 42
    // val := <-ch // Compile error! Cannot receive from send-only channel
}

func receiver(ch <-chan int) {
    // Can only receive from ch
    val := <-ch
    fmt.Println(val)
    // ch <- 1 // Compile error! Cannot send to receive-only channel
}

func main() {
    ch := make(chan int)
    go sender(ch)
    receiver(ch)
}
```

Using directional channels in function signatures makes your code safer and more readable. It tells the reader exactly what the function intends to do with the channel.

## The Select Statement

The **select** statement lets you wait on multiple channel operations at the same time. It is like a switch statement but for channels:

```go
select {
case msg1 := <-ch1:
    fmt.Println("Received from ch1:", msg1)
case msg2 := <-ch2:
    fmt.Println("Received from ch2:", msg2)
case ch3 <- 42:
    fmt.Println("Sent to ch3")
default:
    fmt.Println("No channel ready")
}
```

`select` picks whichever channel operation is ready first. If multiple are ready, it picks one at random. If none are ready, it blocks (unless there is a `default` case).

A common pattern is using select with a timeout:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)

    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()

    select {
    case result := <-ch:
        fmt.Println("Got result:", result)
    case <-time.After(1 * time.Second):
        fmt.Println("Timed out!")
    }
}
```

This prints "Timed out!" because the goroutine takes 2 seconds but we only wait 1 second. The `time.After` function returns a channel that sends the current time after the specified duration.

## Channel Patterns

### Signal Channel

A channel with no data, just used to signal an event:

```go
done := make(chan struct{})

go func() {
    fmt.Println("Working...")
    time.Sleep(time.Second)
    done <- struct{}{} // Signal completion
}()

<-done // Wait for signal
fmt.Println("Done!")
```

### Pipeline

Connecting goroutines in a chain, where the output of one is the input of the next:

```go
package main

import "fmt"

func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generate -> square -> print
    for result := range square(generate(1, 2, 3, 4, 5)) {
        fmt.Println(result) // Prints 1, 4, 9, 16, 25
    }
}
```

### Fan-out / Fan-in

**Fan-out**: Multiple goroutines read from the same channel. **Fan-in**: Multiple channels feed into one channel.

```go
package main

import (
    "fmt"
    "sync"
)

func fanOut(in <-chan int, numWorkers int) []<-chan int {
    channels := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        channels[i] = worker(in)
    }
    return channels
}

func worker(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                out <- val
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    in := make(chan int)
    go func() {
        for i := 1; i <= 10; i++ {
            in <- i
        }
        close(in)
    }()

    workers := fanOut(in, 3)
    for result := range fanIn(workers...) {
        fmt.Println(result)
    }
}
```

## When "Share Memory by Communicating" Clicked

There is a famous Go proverb: **"Do not communicate by sharing memory; share memory by communicating."** For the longest time, I read this and nodded but did not really feel it.

Then I wrote a program where two goroutines needed to update a shared counter. My first instinct was to use a mutex. It worked, but the code felt awkward. Then I rewrote it using a channel where one goroutine owned the counter and others sent increment requests through a channel. The channel version was cleaner, easier to reason about, and had no possibility of forgetting to unlock a mutex.

That is when it clicked. Channels are not just pipes for data. They are a way of thinking about concurrency where data ownership is clear. Only one goroutine owns the data at any time, and the transfer of ownership happens through the channel. No shared state, no locks, no races.

Channels are not the answer to every concurrency problem, but they are a beautiful and powerful tool. Let us keep learning.
