---
title: "Borrowing and References: The Aliasing Invariant"
date: 2025-05-10 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, borrowing, references, memory-safety, invariants]
description: "Rust's borrowing rules encode a fundamental invariant: you can have many readers or one writer, but never both. This single rule eliminates data races at compile time."
---

In Rust, every value has exactly one owner. When the owner goes out of scope, the value is dropped. But real programs need to *use* values without taking ownership — read a string without consuming it, pass data to a function without giving it away.

This is what borrowing does. A **reference** is a pointer to a value that you don't own. You borrow it, use it, and the owner keeps it when you're done. But borrowing isn't a free-for-all. It's governed by two invariants the compiler enforces at every reference boundary.

## The Rules of References

1. **At any given time, you can have either one mutable reference OR any number of immutable references** — but not both.
2. **References must always be valid** — no dangling references, ever.

These two rules are the **aliasing invariant**: shared access and mutable access never coexist. This single constraint eliminates data races at compile time.

## The Problem Borrowing Solves

Consider this code:

```rust
fn main() {
    let the_string = String::from("Hello world");
    let length = calculate_length(the_string);

    println!("{}:{}", the_string, length); // ERROR
}

fn calculate_length(string: String) -> usize {
    string.len()
}
```

The compiler rejects this:

```
error[E0382]: borrow of moved value: `the_string`
 --> src/main.rs:4:23
  |
2 |     let the_string = String::from("Hello world");
  |         ---------- move occurs because `the_string` has type `String`
3 |     let length = calculate_length(the_string);
  |                                   ---------- value moved here
4 |     println!("{}:{}", the_string, length);
  |                       ^^^^^^^^^^ value borrowed here after move
```

Passing `the_string` to `calculate_length` transfers ownership. The function now owns it, and when it returns, the value is dropped. `main` can no longer use it. The ownership invariant is working correctly — but we just wanted to measure the string's length, not consume it.

## References

The solution: pass a *reference* instead of the value itself.

```rust
fn main() {
    let the_string = String::from("Hello world");
    let length = calculate_length(&the_string);

    println!(
        "The string is {}: The length of the string is {}",
        the_string, length
    );
}

fn calculate_length(string: &String) -> usize {
    string.len()
}
```

The `&` creates a reference — a pointer that borrows the value without taking ownership. The function can read the data, but it doesn't own it. When the function returns, the reference disappears, but the original value remains with its owner.

```
The string is Hello world: The length of the string is 11
```

The invariant: **a reference never outlives the value it points to**. The compiler tracks lifetimes and rejects any code where a reference could become dangling.

## Mutable References

Immutable references let you read. Mutable references let you modify:

```rust
fn main() {
    let mut the_string = String::from("Hello world");

    println!("String is: {}", the_string);

    mutate(&mut the_string);

    println!("Updated string is: {}", the_string);
}

fn mutate(string: &mut String) {
    string.push_str(" mutated");
}
```

Output:

```
String is: Hello world
Updated string is: Hello world mutated
```

The `&mut` creates a mutable reference. The function can modify the borrowed value. But here's where the aliasing invariant bites:

## The One-Writer Invariant

You cannot have two mutable references to the same value at the same time:

```rust
fn main() {
    let mut s = String::from("Hello world");

    let r1 = &mut s;
    let r2 = &mut s; // ERROR

    println!("{}, {}", r1, r2);
}
```

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
7 |     println!("{}, {}", r1, r2);
  |                        -- first borrow later used here
```

Why? Because two mutable references to the same data means two parts of the code can modify it simultaneously. In a concurrent context, that's a data race. In a sequential context, it's still a source of bugs — one mutation can invalidate assumptions the other mutation relies on.

The invariant: **at most one mutable reference exists at any point in time**.

## The Readers-Writer Invariant

You also cannot mix immutable and mutable references:

```rust
let mut s = String::from("hello");

let r1 = &s;     // immutable borrow
let r2 = &s;     // another immutable borrow — fine
let r3 = &mut s; // mutable borrow — ERROR

println!("{}, {}, and {}", r1, r2, r3);
```

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:14
  |
4 |     let r1 = &s;
  |              -- immutable borrow occurs here
6 |     let r3 = &mut s;
  |              ^^^^^^ mutable borrow occurs here
8 |     println!("{}, {}, and {}", r1, r2, r3);
  |                                -- immutable borrow later used here
```

The invariant spelled out:

- **Many `&T` (shared/immutable references)** — multiple readers, no writers
- **One `&mut T` (exclusive/mutable reference)** — one writer, no readers

This is the same readers-writer lock pattern used in concurrent systems, but enforced at compile time with zero runtime cost.

## Non-Lexical Lifetimes

The borrow checker is smart about *when* references are actually used:

```rust
fn main() {
    let mut s = String::from("Hello world");

    let r1 = &s;
    println!("{}", r1);
    // r1 is no longer used after this point

    let r2 = &mut s;
    r2.push_str(" Mutated");
    println!("{}", r2);

    let r3 = &s;
    let r4 = &s;
    println!("{}, {}", r3, r4);
}
```

```
Hello world
Hello world Mutated
Hello world Mutated, Hello world Mutated
```

This compiles because the immutable borrow `r1` is no longer alive when the mutable borrow `r2` begins. The compiler tracks the actual usage of references, not just their lexical scope. The invariant holds — there's never a moment when both a mutable and immutable reference coexist.

## The Dangling Reference Invariant

Rust guarantees that references are never dangling:

> If you have a reference to some data, the compiler ensures that the data will not go out of scope before the reference to the data does.

This is enforced through the **lifetime system**. Every reference has a lifetime — the region of code where it's valid. If a reference would outlive the data it points to, the compiler rejects it. No null pointer exceptions. No segfaults from freed memory. The invariant is absolute.

## The Borrowing Invariants, Summarized

| Rule | Invariant |
|:--|:--|
| One mutable reference at a time | No concurrent mutation, no data races |
| Many immutable references allowed | Shared reads are safe |
| No mixing mutable and immutable | Readers never see inconsistent state |
| References cannot outlive their data | No dangling pointers |

These four rules, checked at compile time, eliminate the entire class of memory-related bugs that plague systems programming. The trade-off is that you have to structure your code to satisfy the borrow checker. The benefit is that once it compiles, these bugs are gone — not "unlikely," not "tested against," but *impossible*.

---

*Borrowing and references are how Rust achieves shared access without shared ownership. The aliasing invariant — many readers or one writer, never both — is the foundation of Rust's concurrency safety. Next up: structs, and how to build your own types with their own invariants.*
