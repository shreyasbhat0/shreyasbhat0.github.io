---
title: "Rust Functions: Invariants at Every Boundary"
date: 2025-04-12 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, functions, expressions, statements, invariants]
description: "Every function in Rust is a boundary — and every boundary is a place where invariants are declared. Parameter types, return types, and the expression system all enforce contracts the compiler checks."
---

Functions are boundaries. Every function defines a contract: what it accepts, what it returns, and what must be true for it to work correctly. In most languages, this contract is informal — documented in comments, tested in unit tests, violated in production. In Rust, the contract is the type signature, and the compiler enforces it.

## Functions in Rust

The entry point of every Rust program is `fn main()`. Functions are defined with the `fn` keyword, followed by a name and parentheses:

```rust
fn main() {
    // statements
}
```

Rust uses `snake_case` for function and variable names by convention.

**Defining a function:**

```rust
fn function_name() {
    println!("The Missing Semicolon");
}
```

The invariant at the call site: **if a function takes no parameters and returns nothing, that is exactly what happens**. No hidden side effects from unexpected arguments. No surprise return values. The signature is the truth.

## Function Parameters

Parameters are variables declared in a function's signature. The concrete values you pass are arguments.

```rust
fn main() {
    function_with_parameters("Good Morning");
}

fn function_with_parameters(greetings: &str) {
    println!("Hello, {greetings}");
}
```

Every parameter requires a type annotation. This is not optional. The invariant: **the caller and the callee agree on types at compile time**. You cannot pass an integer where a string is expected. You cannot pass two arguments where one is expected. The function boundary is a checkpoint, and the compiler is the guard.

## Functions With Return Values

Functions declare their return type with `->`:

```rust
fn main() {
    let r = add(5, 10);
    println!("Returned Value: {r}");
}

fn add(x: i32, y: i32) -> i32 {
    x + y // returned implicitly — no semicolon
}
```

Rust offers two ways to return values:

1. **Implicitly** — the last expression in the function body (without a semicolon) is the return value.
2. **Explicitly** — using the `return` keyword.

```rust
fn another_function(x: i32) -> i32 {
    let y = 10 * x;
    return y; // explicit return
}
```

The invariant: **the return type in the signature must match the actual type returned by the body**. If the signature says `-> i32`, the function must return an `i32`. If it doesn't, the program doesn't compile. There is no "returns `undefined`" or "returns `null` sometimes." The type promise is absolute.

## Nested Functions

Rust allows defining functions inside other functions:

```rust
fn main() {
    function();
}

fn function() {
    println!("Function");

    fn inner_function() {
        println!("Nested Function 1");

        fn inner_function2() {
            println!("Nested Function 2");
        }

        inner_function2();
    }

    inner_function();
}
```

Output:

```
Function
Nested Function 1
Nested Function 2
```

Inner functions do **not** have access to the outer scope. They are regular functions that happen to be defined inside another function — they cannot capture variables from the enclosing environment. This is a scope invariant: **a nested function's behavior depends only on its parameters, not on the state of its parent**. If you need to capture the environment, you use closures (a different concept with different rules).

## Statements and Expressions

Function bodies are made of statements and expressions. Understanding the distinction is critical because it governs what returns values and what doesn't.

**Statements** perform actions and do not return values:

```rust
let y = 6; // this is a statement
```

**Expressions** evaluate to a value:

```rust
fn main() {
    let x = {
        let y = 3;
        y + 1
    }; // the block is an expression that evaluates to 4
    println!("{x}"); // prints 4
}
```

Notice that `y + 1` does **not** have a semicolon. Adding a semicolon turns an expression into a statement, which returns `()` (the unit type) instead of a value. This is a subtle but important invariant: **the presence or absence of a semicolon changes the type of a block**.

```rust
fn five() -> i32 {
    5     // expression — returns i32
}

fn oops() -> i32 {
    5;    // statement — returns () — COMPILE ERROR
}
```

The compiler catches this:

```
error[E0308]: mismatched types
  expected `i32`, found `()`
```

This invariant makes the return semantics explicit. There is no guessing about whether a function "fell through" without returning. If the types don't match, the code doesn't compile.

## The Function Boundary as Invariant

Every function in Rust declares a set of invariants through its signature:

| Signature Element | Invariant |
|:--|:--|
| Parameter types | Caller must provide exactly these types |
| Return type | Body must produce exactly this type |
| No return type (`-> ()`) | Function returns nothing; cannot be used as a value |
| `&self` | Method borrows the instance immutably |
| `&mut self` | Method borrows the instance mutably |

These boundaries are checked at every call site. A function that expects `(i32, &str)` and returns `bool` will always receive an `i32` and a `&str`, and will always return a `bool`. This is not "usually true" — it is always true. The compiler guarantees it.

---

*Functions are where Rust's type invariants become most visible. Every parameter, every return type, every expression is a contract. The next post covers ownership — the invariant system that makes Rust unique.*
