# Chapter 8: Authentication & Security

## Why Authentication Matters

Every real application needs authentication. If you are building a side project, you might skip it. I used to skip auth all the time. Just let anyone hit the endpoint, right? Who cares, it is just a demo.

But the moment you build something real, everything changes. You need to know **who** is making a request. You need to protect user data. You need to make sure that user A cannot access user B's orders. You need to make sure that an anonymous visitor cannot delete your entire product catalog.

**Authentication** answers the question: "Who are you?"

**Authorization** answers the question: "What are you allowed to do?"

Both are critical. Without auth, your API is an open door. Anyone can do anything. That might be fine for a public read-only API, but it is never fine for a real application with user data.

## My Personal Lesson

I remember building my first real Go API. It was a simple task manager. I skipped auth completely because I wanted to ship fast. Then I deployed it. Within a day, someone found the API and started creating thousands of fake tasks. The database was full of garbage. That was the moment I realized: **auth is not optional, it is essential**.

It is not just about security either. Auth gives you context. When you know who the user is, you can personalize their experience. You can show their orders, their settings, their data. Without auth, every request is anonymous and every user sees the same thing.

## Common Authentication Approaches

There are several ways to handle authentication in web applications:

**Session-based auth** - The server creates a session after login and stores it in memory or a database. The client gets a session ID in a cookie. Each request sends the cookie, and the server looks up the session. This is simple but does not scale well across multiple servers because sessions need to be shared or synchronized.

**Token-based auth (JWT)** - The server creates a signed token after login. The client stores the token and sends it with each request. The server verifies the token without needing to store anything. This is stateless and scales easily across multiple servers.

**OAuth2** - A protocol that lets users log in through a third party like Google or GitHub. Your application never sees the user's password. This is great for user convenience but adds complexity.

For Go APIs, **JWT is the most common approach**. It is simple, stateless, and works well with microservices. That is what we will focus on in this section.

## A Typical Auth Flow

Here is what the authentication flow looks like in a typical Go API:

```
1. Client sends POST /api/login with email and password
2. Server verifies credentials against the database
3. Server creates a JWT and sends it back
4. Client stores the JWT (localStorage or cookie)
5. Client sends JWT in Authorization header with every request
6. Server middleware verifies the JWT
7. If valid, the request proceeds. If invalid, return 401 Unauthorized.
```

This flow is used by almost every modern API. Once you understand it, you can implement auth for any application.

## What This Section Covers

In this section, I will walk through the fundamentals of authentication in Go:

1. **JWT Authentication** - We will learn what JSON Web Tokens are, how they work, and how to implement them in Go. JWT is the most common way to handle auth in modern APIs. We will create tokens, verify tokens, and build a complete login flow.

2. **Auth Middleware** - We will build middleware that checks every request for a valid JWT. This is how you protect your routes and make sure only authenticated users can access certain endpoints. We will also learn how to pass user information through the request context.

These two topics give you the foundation for adding auth to any Go API. Once you understand JWT and middleware, you can secure any application.

## A Note on Security

Security is a deep topic. This section covers the basics you need to get started. There is much more to learn: **OAuth2**, **session-based auth**, **rate limiting**, **CORS**, **HTTPS**, **input validation**, **SQL injection prevention**, and more.

But you have to start somewhere. JWT and auth middleware are the most practical starting point. They are what you will use in almost every Go API you build.

## Security Best Practices to Keep in Mind

Even though we are only covering JWT and middleware, here are some security practices you should always follow:

- **Always use HTTPS** in production. Never send tokens or passwords over plain HTTP.
- **Hash passwords with bcrypt**. Never store plain-text passwords. Never use MD5 or SHA for passwords.
- **Validate all input**. Never trust data from the client. Check types, lengths, and formats.
- **Use parameterized queries**. Prevent SQL injection by using placeholders like `$1` and `$2`.
- **Set appropriate CORS headers**. Control which domains can call your API.
- **Rate limit your endpoints**. Prevent brute force attacks on login and registration.
- **Keep secrets out of code**. Use environment variables for passwords and keys. Never commit secrets to version control.
- **Set token expiration**. JWTs that never expire are a security risk. Always set a reasonable expiration time.
- **Use strong secret keys**. A weak JWT secret can be brute-forced, allowing attackers to forge valid tokens.

Security is not a feature you add at the end. It is a habit you practice from the beginning. Start with these basics, and you will be ahead of most beginners.

So let us dive in and learn how to protect our applications.
