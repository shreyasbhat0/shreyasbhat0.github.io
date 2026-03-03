---
title: "Closures and Iterators: Zero-Cost Functional Invariants"
date: 2026-03-24 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, closures, iterators, functional-programming, invariants]
description: "Closures capture their environment under the same ownership rules as everything else in Rust. Iterators are lazy, composable, and compile down to the same code as hand-written loops. Functional style, systems performance."
---

In the functions post, we saw that nested functions cannot capture variables from their enclosing scope. That's a deliberate limitation. Closures are the answer — anonymous functions that *can* capture their environment. Combined with iterators, they give Rust a functional programming style that compiles down to the exact same machine code as a hand-written `for` loop. No runtime overhead. No garbage collector. No compromise.

## Closures

A closure is an anonymous function you can save in a variable or pass as an argument. Unlike regular functions, closures can capture values from the scope where they are defined.

```rust
fn main() {
    let greeting = String::from("Hello");

    let say_hello = |name: &str| {
        println!("{}, {}!", greeting, name);
    };

    say_hello("Alice");
    say_hello("Bob");
}
```

```
Hello, Alice!
Hello, Bob!
```

The closure `say_hello` captures `greeting` from the enclosing scope. A regular function cannot do this — try it and the compiler stops you. Closures use pipes `|...|` instead of parentheses for their parameters, and the body can be a single expression or a block.

Type annotations are optional on closures. The compiler infers them from how you use the closure:

```rust
fn main() {
    let add = |x, y| x + y;
    let result = add(5, 10);
    println!("{}", result); // 15
}
```

But here is a subtle invariant — once the compiler infers the types, they are fixed. You cannot call the same closure with different types:

```rust
fn main() {
    let identity = |x| x;

    let a_string = identity(String::from("hello"));
    let a_number = identity(42); // ERROR
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:5:28
  |
5 |     let a_number = identity(42);
  |                    -------- ^^ expected `String`, found integer
  |                    |
  |                    arguments to this function are incorrect
```

The first call locked the type to `String`. The second call tries `i32`. The compiler rejects it. The closure's type signature is an invariant — inferred once, enforced everywhere.

## How Closures Capture: Fn, FnMut, FnOnce

This is where it gets interesting. Closures capture variables from their environment, but *how* they capture is governed by the same ownership rules as everything else in Rust. The compiler looks at what the closure does with the captured variable and picks the least restrictive mode:

**Immutable borrow (`Fn`)** — the closure only reads the captured variable:

```rust
fn main() {
    let the_secret_message = String::from("Rust is fun");

    let read_it = || println!("{}", the_secret_message);

    read_it();
    read_it();
    println!("Still here: {}", the_secret_message); // works fine
}
```

The closure borrows `the_secret_message` immutably. You can still use the original variable after the closure, and you can call the closure multiple times.

**Mutable borrow (`FnMut`)** — the closure modifies the captured variable:

```rust
fn main() {
    let mut shopping_list = vec![String::from("milk")];

    let mut add_item = |item: &str| {
        shopping_list.push(String::from(item));
    };

    add_item("eggs");
    add_item("bread");

    println!("{:?}", shopping_list); // ["milk", "eggs", "bread"]
}
```

The closure mutably borrows `shopping_list`. Notice the closure itself must be declared `mut` — because calling it changes state. While the mutable borrow is active, you cannot use `shopping_list` directly. The aliasing invariant from the borrowing post applies here too: one writer, no readers.

**Move (`FnOnce`)** — the closure takes ownership of the captured variable:

```rust
fn main() {
    let farewell = String::from("goodbye");

    let consume_it = move || {
        println!("{}", farewell);
        drop(farewell);
    };

    consume_it();
    // consume_it(); // ERROR: closure has been consumed
    // println!("{}", farewell); // ERROR: value moved
}
```

The `move` keyword forces the closure to take ownership. After calling `consume_it()`, the value is gone. The closure consumes both the captured value and itself — it can only be called once. That is what `FnOnce` means.

These three traits form a hierarchy: every `Fn` closure is also `FnMut`, and every `FnMut` closure is also `FnOnce`. The compiler picks the most permissive trait the closure qualifies for. You do not annotate this — the compiler infers it from what the closure body does. The ownership invariants you already know are doing the work.

## Iterators

Iterators are Rust's way of processing sequences of values. The core idea is the `Iterator` trait:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

That is the entire contract. An iterator produces values one at a time via `next()`, returning `Some(value)` until the sequence is exhausted, then returning `None`. Every iterator in Rust — over vectors, strings, hash maps, files — follows this one trait.

The simplest way to get an iterator:

```rust
fn main() {
    let flavors = vec!["vanilla", "chocolate", "strawberry"];
    let mut iter = flavors.iter();

    println!("{:?}", iter.next()); // Some("vanilla")
    println!("{:?}", iter.next()); // Some("chocolate")
    println!("{:?}", iter.next()); // Some("strawberry")
    println!("{:?}", iter.next()); // None
}
```

After the last element, every subsequent call returns `None`. That is the iterator invariant: **once `None` is returned, the sequence is done**.

## Lazy Evaluation

Here is a critical property: iterators in Rust are **lazy**. Nothing happens until you consume them.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // This does NOTHING yet
    let doubled = numbers.iter().map(|x| x * 2);

    println!("{:?}", doubled);
}
```

```
Map { iter: Iter([1, 2, 3, 4, 5]) }
```

No values were doubled. The `map` call just created a new iterator that *will* double values when consumed. You need a **consuming adaptor** like `collect`, `for_each`, `sum`, or `fold` to actually run the chain:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    println!("{:?}", doubled); // [2, 4, 6, 8, 10]
}
```

The compiler even warns you if you forget to consume an iterator:

```
warning: unused `Map` that must be used
 --> src/main.rs:4:5
  |
4 |     numbers.iter().map(|x| x * 2);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
```

The invariant: **iterator adaptors do not execute until consumed**. This means you can build up a chain of transformations without allocating intermediate collections or doing any work — and then execute the entire pipeline in one pass.

## Common Iterator Adaptors

Let's build up a practical example step by step. Say you have a list of student scores and you want to find the average of all passing grades (above 60).

Start with the data:

```rust
fn main() {
    let scores = vec![85, 42, 93, 58, 71, 36, 99, 67];
```

**Filter** — keep only passing scores:

```rust
    let passing = scores.iter().filter(|&&score| score > 60);
```

**Map** — convert to `f64` for averaging:

```rust
    let as_floats = passing.map(|&score| score as f64);
```

**Fold** — accumulate the sum and count in a single pass:

```rust
    let (sum, count) = as_floats.fold((0.0, 0), |(sum, count), score| {
        (sum + score, count + 1)
    });

    if count > 0 {
        println!("Average passing score: {:.1}", sum / count as f64);
    }
}
```

```
Average passing score: 83.0
```

Now chain the whole thing together:

```rust
fn main() {
    let scores = vec![85, 42, 93, 58, 71, 36, 99, 67];

    let (sum, count) = scores
        .iter()
        .filter(|&&s| s > 60)
        .map(|&s| s as f64)
        .fold((0.0, 0), |(sum, count), s| (sum + s, count + 1));

    if count > 0 {
        println!("Average passing score: {:.1}", sum / count as f64);
    }
}
```

No intermediate vectors. No temporary allocations. One pass through the data. The compiler optimizes this chain into a tight loop — equivalent to what you would write by hand with a `for` loop, a running sum, and a counter. This is what "zero-cost abstraction" means: the functional style does not cost you anything at runtime.

Some other adaptors worth knowing:

```rust
fn main() {
    let words = vec!["hello", "world", "rust"];

    // collect into a new collection
    let upper: Vec<String> = words.iter().map(|w| w.to_uppercase()).collect();
    println!("{:?}", upper); // ["HELLO", "WORLD", "RUST"]

    // sum
    let total: i32 = vec![1, 2, 3, 4].iter().sum();
    println!("{}", total); // 10

    // enumerate — gives you the index
    for (i, word) in words.iter().enumerate() {
        println!("{}: {}", i, word);
    }

    // any / all — short-circuit checks
    let has_long_word = words.iter().any(|w| w.len() > 4);
    println!("Has long word: {}", has_long_word); // true
}
```

## The Zero-Cost Invariant

Here is the invariant that ties closures and iterators together: **functional-style code in Rust compiles to the same machine code as imperative loops**.

Consider these two versions:

```rust
// Imperative
let mut sum = 0;
for i in 0..1000 {
    if i % 2 == 0 {
        sum += i * i;
    }
}

// Functional
let sum: i32 = (0..1000)
    .filter(|i| i % 2 == 0)
    .map(|i| i * i)
    .sum();
```

Both produce the same result. Both compile to the same assembly. The iterator version is not an abstraction you pay for — it is an abstraction the compiler erases. The chain of closures gets inlined, the iterator state machine gets flattened, and what remains is a loop with a conditional and an accumulator. You get readability and performance. No trade-off.

This works because closures in Rust are not heap-allocated objects with vtables (like in many other languages). Each closure is a unique, anonymous struct that the compiler can see through and inline. Iterators are similarly concrete types that the optimizer dissolves into straight-line code.

## Closures and Iterators, Summarized

| Concept | Invariant |
|:--|:--|
| Closure type inference | Types are fixed after first use |
| `Fn` | Captures by immutable borrow — callable many times |
| `FnMut` | Captures by mutable borrow — callable many times, mutates state |
| `FnOnce` | Captures by move — callable once, consumes captured values |
| Iterator laziness | No work happens until the chain is consumed |
| Zero-cost abstraction | Iterator chains compile to the same code as hand-written loops |

---

*Closures and iterators bring functional programming to Rust without sacrificing the performance guarantees that make Rust a systems language. The ownership rules you already know govern how closures capture, and the optimizer ensures iterator chains cost nothing. Next up: modules and crates — how Rust organizes code and enforces visibility boundaries.*
