# WaitGroup Internals

## Why Look Under the Hood?

You might wonder: why do I need to know how WaitGroup works internally? I had the same thought. But then I hit a bug where `wg.Wait()` never returned, and I had no idea why. Understanding the internals helped me figure out that I was calling `Add` with a negative number by accident.

Knowing how WaitGroup works under the hood helps you:

- Debug tricky synchronization bugs
- Understand why certain patterns are correct or incorrect
- Appreciate the efficiency of Go's standard library
- Avoid misuse that can cause panics or deadlocks

## The Core Idea: An Atomic Counter

At its heart, WaitGroup is an **atomic counter** with some clever blocking mechanics. No mutex is needed for the counter itself because it uses atomic operations, which are way faster than lock-based synchronization.

Here is the basic idea:

- `Add(n)` increments the counter by n
- `Done()` decrements the counter by 1 (it is literally `Add(-1)`)
- `Wait()` blocks until the counter reaches 0

But there is more to it than just a counter. WaitGroup also needs to track how many goroutines are waiting and efficiently wake them all up when the counter reaches zero.

## The State Field

Internally, WaitGroup uses a single 64-bit integer to store two pieces of information:

- **High 32 bits**: the counter (how many goroutines are still running)
- **Low 32 bits**: the waiter count (how many goroutines are blocked in Wait())

```
|----------------|----------------|
|   Counter (32) | Waiters (32)   |
|----------------|----------------|
```

Using a single 64-bit value allows both the counter and waiter count to be updated atomically in one operation. This avoids race conditions between incrementing the counter and checking if there are waiters.

Let me visualize this:

```
State value: 0x0000000200000001
  Counter:  0x00000002 = 2 (two goroutines still running)
  Waiters:  0x00000001 = 1 (one goroutine waiting in Wait())
```

## How Add Works

When you call `wg.Add(n)`, here is what happens:

1. Atomically add n to the high 32 bits (the counter)
2. If the counter becomes 0 and there are waiters, release all waiters using a semaphore
3. If the counter goes negative, panic! This means you called Done() more times than Add()

```go
// Simplified pseudo-code
func (wg *WaitGroup) Add(delta int) {
    state := atomic.AddUint64(&wg.state, uint64(delta)<<32)

    counter := int32(state >> 32)
    waiters := uint32(state)

    if counter < 0 {
        panic("sync: WaitGroup is reused before previous Wait has returned")
    }

    if counter == 0 && waiters > 0 {
        // Wake up all waiters
        for i := 0; i < int(waiters); i++ {
            runtime_Semrelease(&wg.sema)
        }
    }
}
```

The panic on negative counter is a safety measure. It catches the bug where you call `Done()` more times than `Add()`, which would otherwise silently corrupt the counter.

## How Done Works

`Done()` is literally just `Add(-1)`:

```go
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

That is it. Nothing more. When the counter decrements to zero and there are waiters, the Add function handles waking them up.

## How Wait Works

When you call `wg.Wait()`, here is what happens:

1. Read the current state atomically
2. If the counter is already 0, return immediately
3. If the counter is not 0, increment the waiter count and block on a semaphore
4. When woken up (by Add when counter reaches 0), return

```go
// Simplified pseudo-code
func (wg *WaitGroup) Wait() {
    for {
        state := atomic.LoadUint64(&wg.state)
        counter := int32(state >> 32)

        if counter == 0 {
            return // No goroutines to wait for
        }

        // Try to increment waiter count
        if atomic.CompareAndSwapUint64(&wg.state, state, state+1) {
            runtime_Semacquire(&wg.sema) // Block until woken
            return
        }
        // If CAS failed, another goroutine changed the state. Retry.
    }
}
```

The **compare-and-swap (CAS)** operation is crucial. It ensures that the waiter count is only incremented if no other goroutine changed the state between our load and our increment. This is lock-free programming at work.

## A Simplified Implementation

Here is a simplified version that shows the key ideas. This is not the real implementation but captures the essence:

```go
package main

import (
    "fmt"
    "sync/atomic"
    "unsafe"
)

type SimpleWaitGroup struct {
    state int64 // high 32 = counter, low 32 = waiters
    sema  uint32
}

func (wg *SimpleWaitGroup) Add(delta int) {
    newState := atomic.AddInt64(&wg.state, int64(delta)<<32)
    counter := int32(newState >> 32)
    waiters := uint32(newState)

    if counter < 0 {
        panic("negative counter")
    }

    if counter == 0 && waiters > 0 {
        // In real implementation, this uses runtime_Semrelease
        // to wake up all waiting goroutines
        fmt.Printf("Waking up %d waiters\n", waiters)
    }
}

func (wg *SimpleWaitGroup) Done() {
    wg.Add(-1)
}

func (wg *SimpleWaitGroup) Wait() {
    for {
        state := atomic.LoadInt64(&wg.state)
        counter := int32(state >> 32)

        if counter == 0 {
            return
        }

        // Try to increment waiter count
        if atomic.CompareAndSwapInt64(&wg.state, state, state+1) {
            // In real implementation, this uses runtime_Semacquire
            // to block until woken up
            _ = unsafe.Pointer(nil) // placeholder
            return
        }
    }
}

func main() {
    var wg SimpleWaitGroup
    wg.Add(3)
    fmt.Printf("Counter after Add(3): %d\n", int32(atomic.LoadInt64(&wg.state)>>32))

    wg.Done()
    fmt.Printf("Counter after Done(): %d\n", int32(atomic.LoadInt64(&wg.state)>>32))

    wg.Done()
    wg.Done()
    fmt.Printf("Counter after 2 more Done(): %d\n", int32(atomic.LoadInt64(&wg.state)>>32))
}
```

## Why No Mutex for the Counter?

You might wonder why WaitGroup uses atomic operations instead of a simple mutex. The answer is **performance**.

Atomic operations are much faster than mutex operations for simple counter updates:

- **Mutex**: acquire lock, update counter, release lock. This involves at least two atomic operations plus potential thread descheduling.
- **Atomic**: one CPU instruction to update the counter. No lock contention, no waiting, no descheduling.

When you have thousands of goroutines calling Done() at nearly the same time, the difference matters. Atomic operations scale much better than mutex-based approaches.

## The Reuse Panic

One more important internal detail. WaitGroup has a check to prevent reuse while Wait() is still running. If you call Add() with a positive delta while a Wait() call is still in progress, you get a panic:

```
panic: sync: WaitGroup is reused before previous Wait has returned
```

This happens because the real WaitGroup resets the state to zero after waking up waiters. If you try to add new goroutines before all waiters have been woken and have returned, the state could become inconsistent.

The fix is simple: do not reuse a WaitGroup while someone might still be in Wait(). Create a new one instead.

## Key Takeaways

Understanding WaitGroup internals taught me several things:

- **Atomic operations are powerful**. They let you update shared state without locks, which is faster and often simpler.
- **Bit manipulation is common in low-level code**. Storing two values in one 64-bit integer to enable atomic updates of both together is a standard technique.
- **CAS (compare-and-swap) is the building block** of lock-free algorithms. The Wait loop uses CAS to safely increment the waiter count.
- **Panics are a safety net**. The panic on negative counter and the panic on reuse catch bugs early instead of letting them corrupt state silently.

The next time you write `wg.Add(1)` and `defer wg.Done()`, you will know exactly what is happening under the surface. And when something goes wrong, you will have the tools to figure out why.
