+++
isCJKLanguage = true
title = "Simplify Gin CRUD Endpoints Using Go Generics (With Code Examples)"
description = "Learn how to eliminate boilerplate in Gin by using Go generics to create reusable, type-safe CRUD handlers. This guide provides a practical example of refactoring repetitive API endpoint logic into a single generic function, resulting in cleaner and more maintainable code."
keywords = ["Go", "Golang", "Gin", "Go Generics", "CRUD", "API", "REST API", "Refactoring", "Boilerplate", "Code Simplification", "Web Development", "Type-Safe"]
date = 2025-09-16T15:20:00+08:00
authors = ["muzhy"]
tags = ["Go", "Generics", "Gin"]
categories = ["Go"]
cover = "/images/Go_Generics_simply_Gin.png"
draft = false
+++

# Simplify Gin CRUD Endpoints Using Go Generics (With Code Examples)

If you’ve ever built APIs with [Gin](https://gin-gonic.com/), you’ve probably noticed a pattern: every new resource—users, products, orders—comes with its own nearly identical `Create` handler. Copy, paste, tweak the type, repeat. It works, but it’s tedious and error‑prone.

Go’s introduction of **generics** changes the game. Instead of duplicating logic for each resource, we can abstract the common parts into a single, reusable function. The result: cleaner code, fewer bugs, and a much happier developer experience.

---

## The Old Way: Copy‑Paste Handlers

Here’s what a typical `Create` endpoint for an `Account` might look like:

```go
func CreateAccountHandler(c *gin.Context, repo Repo) {
    var account Account
    if err := c.ShouldBindJSON(&account); err != nil {
        Fail(c, pkg.ErrInvalidParam)
        return
    }
    if err := account.Create(repo); err != nil {
        Fail(c, err)
        return
    }
    Success(c, account)
}
```

Not terrible, but when you need the same for `Product`, `Order`, or `Invoice`, you’re stuck duplicating this block over and over. That’s boilerplate begging to be eliminated.

---

## The Generic Way: One Function, Many Resources

With generics, we can define a **type constraint** that ensures any resource we pass in knows how to create itself:

```go
type Creator interface {
    Create(ctx context.Context, repo Repo) error
}
```

Now we can write a single generic function that works for any type implementing `Creator`:

```go
func Create[T Creator](repo Repo, factory func() T) func(c *gin.Context) {
    return func(c *gin.Context) {
        data := factory()
        if err := c.ShouldBindJSON(data); err != nil {
            Fail(c, pkg.ErrInvalidParam)
            return
        }
        ctx := c.Request.Context()
        if err := data.Create(c, repo); err != nil {
            Fail(c, err)
            return
        }
        Success(c, data)
    }
}
```

And here’s the magic: registering a new endpoint is now a **one‑liner**.

```go
router.POST("/accounts", Create(repo, func() *Account { return &Account{} }))
```

Want to add `Product` or `Order`? Just implement `Creator` and reuse the same generic function. No more copy‑paste.

---

## Why This Matters

- **Less boilerplate** → fewer bugs and easier maintenance  
- **Type safety** → the compiler enforces correctness  
- **Scalability** → adding new resources is trivial  

Generics let us treat common patterns as templates, freeing us from repetitive code and letting us focus on actual business logic.

---

## What’s Next?

In this post, we only tackled `Create`. But the same approach works beautifully for `Read`, `Update`, and `Delete`. Imagine a full CRUD API where each endpoint is registered in a single line, powered by generics.