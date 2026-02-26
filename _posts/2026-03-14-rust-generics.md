---
title: "Generics: Write Once, Check Every Type"
date: 2026-03-14 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, generics, type-parameters, monomorphization, invariants]
description: "Generics let you write code that works for many types without sacrificing type safety or performance. The compiler generates specialized code for each concrete type — zero cost, full safety."
---

Consider an example. You need a function that returns the larger of two integers:

```rust
fn largest_i32(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }
}
```

Works great. Now you need the same thing for `f64`:

```rust
fn largest_f64(a: f64, b: f64) -> f64 {
    if a > b { a } else { b }
}
```

And for `i64`:

```rust
fn largest_i64(a: i64, b: i64) -> i64 {
    if a > b { a } else { b }
}
```

Three functions. Same logic. Different types. This is the exact problem generics solve.

## Generic Functions

Instead of duplicating the function for every type, you write it once with a **type parameter**:

```rust
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}
```

`T` is a placeholder for any type. The `PartialOrd` bound says: `T` must be a type that supports comparison with `>`. The compiler checks this at every call site.

```rust
fn main() {
    let result_int = largest(10, 20);
    let result_float = largest(3.14, 2.71);
    println!("{} {}", result_int, result_float); // 20 3.14
}
```

One function, multiple types. The compiler verifies that `i32` implements `PartialOrd` (it does) and that `f64` implements `PartialOrd` (it does). If you try to call `largest` with a type that doesn't support comparison:

```rust
struct Zombie {
    health: i32,
}

fn main() {
    let z1 = Zombie { health: 100 };
    let z2 = Zombie { health: 50 };
    let bigger = largest(z1, z2);
}
```

```
error[E0369]: binary operation `>` cannot be applied to type `Zombie`
 --> src/main.rs:2:10
  |
2 |     if a > b { a } else { b }
  |        - ^ - Zombie
  |        |
  |        Zombie
  |
help: an implementation of `PartialOrd` might be missing for `Zombie`
```

The compiler tells you exactly what's missing. `Zombie` doesn't implement `PartialOrd`, so it can't be compared. The generic function's contract is enforced for every concrete type.

## Generic Structs

Structs can be generic too. Consider a `Wrapper` that holds a value of any type:

```rust
struct Wrapper<T> {
    value: T,
}

fn main() {
    let wrapped_int = Wrapper { value: 42 };
    let wrapped_string = Wrapper { value: String::from("the_missing_semicolon") };

    println!("{}", wrapped_int.value);
    println!("{}", wrapped_string.value);
}
```

One struct definition, works with `i32`, `String`, or anything else. The compiler knows the concrete type at each usage — `wrapped_int` is a `Wrapper<i32>`, `wrapped_string` is a `Wrapper<String>`. Type safety is preserved.

You can use multiple type parameters:

```rust
struct Pair<A, B> {
    first: A,
    second: B,
}

fn main() {
    let p = Pair { first: 1, second: "hello" };
    println!("{} {}", p.first, p.second);
}
```

`Pair<i32, &str>` — the compiler tracks both types independently.

## Generic Enums: You Already Know Them

Here's something that might click: `Option` and `Result` are generic enums. You've been using generics since your first Rust program.

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Option<i32>` can be `Some(42)` or `None`. `Option<String>` can be `Some(String::from("hello"))` or `None`. Same enum, different types, full type safety. When you unwrap an `Option<i32>`, you know you're getting an `i32` — not a string, not null, not undefined. The generic type parameter locks it down.

`Result<User, DatabaseError>` means: this operation either succeeds with a `User` or fails with a `DatabaseError`. The types in the signature tell you exactly what to expect. No surprises.

## Generic Method Implementations

You can implement methods on generic structs:

```rust
struct Wrapper<T> {
    value: T,
}

impl<T> Wrapper<T> {
    fn new(value: T) -> Self {
        Wrapper { value }
    }

    fn unwrap(self) -> T {
        self.value
    }
}

fn main() {
    let w = Wrapper::new(99);
    let inner = w.unwrap();
    println!("{}", inner); // 99
}
```

The `impl<T>` says: these methods exist for `Wrapper` regardless of what `T` is. But you can also implement methods only for specific types:

```rust
impl Wrapper<f64> {
    fn round(&self) -> f64 {
        self.value.round()
    }
}
```

Now `.round()` only exists on `Wrapper<f64>`. Call it on a `Wrapper<String>` and the compiler rejects it. The available methods depend on the concrete type — the compiler tracks this precisely.

## Trait Bounds on Generics

Trait bounds are where generics and traits meet. Without bounds, a generic type parameter tells the compiler almost nothing — you can't call any methods on it because the compiler doesn't know what methods exist:

```rust
fn print_it<T>(value: T) {
    println!("{}", value); // ERROR
}
```

```
error[E0277]: `T` doesn't implement `std::fmt::Display`
 --> src/main.rs:2:20
  |
2 |     println!("{}", value);
  |                    ^^^^^ `T` cannot be formatted with the default formatter
  |
help: consider restricting type parameter `T`
  |
1 | fn print_it<T: std::fmt::Display>(value: T) {
  |              ++++++++++++++++++++
```

The compiler won't assume `T` can be displayed. You have to declare it:

```rust
fn print_it<T: std::fmt::Display>(value: T) {
    println!("{}", value);
}
```

Now the contract is explicit: `print_it` works for any type that implements `Display`. The compiler checks both sides — the function body only uses methods guaranteed by the bound, and every call site provides a type that satisfies the bound.

Multiple bounds work the same way as with traits:

```rust
fn clone_and_print<T: Clone + std::fmt::Debug>(value: &T) {
    let cloned = value.clone();
    println!("{:?}", cloned);
}
```

`T` must be both `Clone` and `Debug`. The compiler verifies both at every call site.

## Monomorphization: The Zero-Cost Part

Here's the part that makes generics in Rust different from generics in most other languages. When you write:

```rust
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

fn main() {
    largest(10_i32, 20_i32);
    largest(3.14_f64, 2.71_f64);
}
```

The compiler doesn't generate one function that handles all types at runtime. It generates **two specialized functions** — one for `i32` and one for `f64`:

```rust
// What the compiler actually produces (conceptually):
fn largest_i32(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }
}

fn largest_f64(a: f64, b: f64) -> f64 {
    if a > b { a } else { b }
}
```

This process is called **monomorphization** — turning polymorphic code into monomorphic (single-type) code. Each call site gets a function specialized for its exact types, with zero indirection and zero runtime dispatch.

The implications are significant:

- **No runtime cost.** Generic code runs exactly as fast as hand-written specialized code.
- **No boxing.** Values aren't wrapped in runtime type containers.
- **Full optimization.** The compiler can inline and optimize each specialization independently.

This is the invariant that makes Rust's generics fundamentally different from, say, Java's. In Java, generics are erased at runtime — `List<Integer>` and `List<String>` become the same `List` at the bytecode level. In Rust, `Vec<i32>` and `Vec<String>` are completely different types with completely different generated code. The type information isn't erased. It's used to generate better code.

The trade-off is compile time and binary size — more specializations mean more code to compile and more machine code in the final binary. But the runtime cost is zero. You don't trade safety for flexibility, and you don't trade flexibility for performance.

## Putting It Together

Let's try this out. A generic function with trait bounds, used with a custom type:

```rust
use std::fmt;

#[derive(Debug, Clone)]
struct Score {
    player: String,
    value: u32,
}

impl fmt::Display for Score {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}: {}", self.player, self.value)
    }
}

impl PartialOrd for Score {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        self.value.partial_cmp(&other.value)
    }
}

impl PartialEq for Score {
    fn eq(&self, other: &Self) -> bool {
        self.value == other.value
    }
}

fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

fn main() {
    let s1 = Score { player: String::from("Alice"), value: 850 };
    let s2 = Score { player: String::from("Bob"), value: 920 };
    let winner = largest(s1, s2);
    println!("Winner: {}", winner);
}
```

```
Winner: Bob: 920
```

`Score` implements `PartialOrd`, so it works with `largest`. The compiler monomorphizes `largest` into a version specialized for `Score`. Type-safe, zero overhead, works with any type that satisfies the bounds.

---

*Generics are how Rust eliminates code duplication without eliminating type safety. You write the logic once, the compiler checks it for every type you use, and monomorphization ensures the generated code is as fast as if you'd written each specialization by hand. No runtime cost. No type erasure. The type-safety invariant holds all the way down to the machine code.*
