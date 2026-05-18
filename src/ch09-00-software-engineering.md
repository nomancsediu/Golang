# Chapter 9: Software Engineering Concepts

## Writing Code is Easy. Writing Maintainable Code is Hard.

Anyone can write code that works. I have written plenty of code that works on the first try. But give it a few weeks, add a few features, and suddenly you cannot understand your own code anymore. Every change breaks something. Every new feature takes longer than the last. The codebase feels like a tangled mess.

That is the difference between **code that works** and **code that lasts**. Software engineering is about writing code that other people (including future you) can understand, change, and extend without fear.

## What This Section Covers

This section is about the concepts that took my code from "it works on my machine" to "this is actually professional quality":

1. **Remove Tight Coupling** - When your components depend on each other too much, everything breaks together. We will learn how to fix this with interfaces and dependency injection.

2. **Interface in Go** - Interfaces are the key to flexible, testable code. Go's approach to interfaces is unique and powerful. We will learn how to use them effectively.

3. **Design Patterns** - Common solutions to common problems. Not every pattern is useful in Go, but some are essential. We will cover the ones that matter most.

4. **Domain Driven Design** - Organizing your code around the business domain instead of technical concerns. This changed how I think about software.

## My Personal Journey

I used to think software engineering concepts were "academic" and not practical. I just wanted to write code that works. But the more I built real applications, the more I realized that these concepts are not academic at all. They are survival skills.

**Tight coupling** is what made my early projects impossible to change. When I changed the database, I had to update 50 files. When I changed the email service, half the app broke. Learning to decouple my code with interfaces was a game changer.

**Design patterns** sounded fancy, but once I started seeing the same problems over and over, I realized that patterns are just names for solutions I was already reinventing poorly.

**Domain Driven Design** was the biggest shift. Instead of thinking "I need a database table and an API endpoint," I started thinking "what does the business need? What are the rules? What are the concepts?" The code got simpler, not more complex.

## The Key Insight

The key insight across all these concepts is the same: **depend on abstractions, not implementations**. When your code depends on interfaces instead of concrete types, you can swap implementations without changing anything else. You can test in isolation. You can evolve each part independently.

This is not just theory. It is practical. Every time I apply these concepts, my code gets easier to work with. Every time I skip them, I regret it later.

Here is a simple example of what I mean:

```go
// Tightly coupled: depends on a concrete type
type OrderService struct {
    db *sql.DB  // Stuck with PostgreSQL
}

// Loosely coupled: depends on an interface
type OrderRepository interface {
    Save(order *Order) error
    FindByID(id int64) (*Order, error)
}

type OrderService struct {
    repo OrderRepository  // Can use any implementation
}
```

The second version is better because `OrderService` does not know or care whether data is stored in PostgreSQL, MongoDB, or memory. It just needs something that can save and find orders. This makes the code testable, flexible, and maintainable.

## A Word of Caution

Do not go overboard. I went through a phase where every struct had an interface, every function was a factory, and every dependency was injected. The code became harder to read, not easier. There were interfaces for things that would never have more than one implementation. There were abstractions for abstractions' sake.

The goal is not to use every pattern. The goal is to write code that you can understand, change, and test. Sometimes that means a simple function. Sometimes that means a carefully designed interface. The trick is knowing when each approach is appropriate.

As a rule of thumb: add abstraction when you feel pain. If changing one thing requires changing five other things, that is pain. If you cannot test a component without spinning up a database, that is pain. If two features are tangled together and you cannot work on one without breaking the other, that is pain. These are the moments when software engineering concepts help.

## How to Read This Section

Each chapter in this section builds on the previous one. Tight coupling introduces the problem. Interfaces provide the solution. Design patterns show common applications. Domain Driven Design brings it all together into a coherent approach.

You do not need to master everything at once. Read through, understand the concepts, and then apply them one at a time. Start with interfaces. They are the most practical and immediately useful concept. Then move to decoupling. Then patterns. Then DDD.

The important thing is not to memorize terminology. The important thing is to develop an intuition for good code structure. When you see tangled code, you should feel it. When you see clean code, you should appreciate it. That intuition comes from practice, not from reading.

## The Go Philosophy

Go was designed with these concepts in mind. Interfaces are implicit because Go wants you to define them at the point of use, not the point of implementation. Composition is preferred over inheritance because it leads to more flexible code. Small, focused interfaces are encouraged because they are easier to understand and implement.

When you learn software engineering concepts in Go, you will find that the language already guides you toward good practices. You just need to follow where it leads.

Let us dive in and learn these concepts one by one.
