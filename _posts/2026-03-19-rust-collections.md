---
title: "Collections: Owned Data with Built-in Guarantees"
date: 2026-03-19 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, collections, vec, hashmap, string, invariants]
description: "Vec, String, and HashMap are the workhorses of Rust. Each one owns its data on the heap and enforces invariants the compiler alone can't — contiguous memory, valid UTF-8, unique keys."
---

So far in this series, every type we've worked with has been fixed-size: integers, booleans, arrays with a known length. Real programs need data structures that grow. Rust's standard library gives you three main ones: `Vec<T>` for ordered sequences, `String` for text, and `HashMap<K, V>` for key-value lookups. Each one owns its data on the heap, and each one enforces guarantees that go beyond what the type system alone provides.

## Vec: A Growable Array

A `Vec<T>` is a contiguous, growable array of elements of type `T`. It owns its data, allocates on the heap, and cleans up when it goes out of scope.

### Creating and Pushing

```rust
fn main() {
    let mut shopping_list: Vec<String> = Vec::new();
    shopping_list.push(String::from("coffee"));
    shopping_list.push(String::from("bread"));
    shopping_list.push(String::from("hot sauce"));

    println!("{:?}", shopping_list);
}
```

```
["coffee", "bread", "hot sauce"]
```

You can also use the `vec!` macro for inline creation:

```rust
let numbers = vec![1, 2, 3, 4, 5];
let all_zeros = vec![0; 10]; // ten zeros
```

### Accessing Elements

Two ways to read from a `Vec`: indexing and `.get()`. They have very different behavior when things go wrong.

```rust
fn main() {
    let flavors = vec!["vanilla", "chocolate", "strawberry"];

    let first = &flavors[0];
    println!("First flavor: {}", first);

    let maybe_tenth = flavors.get(10);
    println!("Tenth flavor: {:?}", maybe_tenth);
}
```

```
First flavor: vanilla
Tenth flavor: None
```

Now let's try indexing out of bounds:

```rust
fn main() {
    let flavors = vec!["vanilla", "chocolate", "strawberry"];
    let oops = &flavors[10]; // PANIC
    println!("{}", oops);
}
```

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 10', src/main.rs:3:17
```

Direct indexing with `[]` panics on an invalid index. It refuses to give you garbage data — if the index is out of bounds, the program stops. The `.get()` method returns an `Option<&T>` instead: `Some(&value)` if the index exists, `None` if it doesn't. No panic, no crash, you handle both cases explicitly.

This is the `Vec` access invariant: **you never silently read invalid memory**. Either you get the right data, or you get a panic (indexing) or a `None` (.get()). Pick the one that fits your situation — `.get()` when the index might be invalid, `[]` when an invalid index is a bug that should crash.

### Iterating

The idiomatic way to walk through a `Vec`:

```rust
fn main() {
    let the_crew = vec!["Alice", "Bob", "Charlie"];

    for member in &the_crew {
        println!("{}", member);
    }

    // the_crew is still usable here — we borrowed, not moved
    println!("Crew size: {}", the_crew.len());
}
```

Iterating with `&the_crew` borrows each element. If you need to modify elements in place:

```rust
fn main() {
    let mut scores = vec![80, 90, 75];

    for score in &mut scores {
        *score += 5; // dereference to modify
    }

    println!("{:?}", scores); // [85, 95, 80]
}
```

## String: Always Valid UTF-8

`String` is Rust's owned, growable, heap-allocated string type. It is not the same as `&str`. This trips up almost everyone coming from other languages.

### String vs &str

- `String` — owned, heap-allocated, mutable, growable. Think of it as `Vec<u8>` that guarantees valid UTF-8.
- `&str` — a borrowed reference to a string slice. Immutable. Often a view into a `String` or a string literal baked into the binary.

```rust
fn main() {
    let owned: String = String::from("I own this data");
    let borrowed: &str = "I'm a string literal"; // &'static str
    let slice: &str = &owned[..5]; // a slice of the owned String

    println!("{}", owned);
    println!("{}", borrowed);
    println!("{}", slice);
}
```

```
I own this data
I'm a string literal
I own
```

The general pattern: use `String` when you need to own or modify text. Use `&str` when you just need to read it. Functions that accept `&str` can take both — a `&String` coerces to `&str` automatically.

### Building Strings

```rust
fn main() {
    let mut greeting = String::from("Hello");
    greeting.push_str(", world");
    greeting.push('!');

    println!("{}", greeting); // Hello, world!

    let first = String::from("Rust");
    let second = String::from(" is fun");
    let combined = first + &second; // first is moved, second is borrowed

    println!("{}", combined); // Rust is fun
    // println!("{}", first); // ERROR: first was moved
}
```

The `+` operator takes ownership of the left side. This is a common gotcha. If you need to combine multiple strings without losing any of them, use `format!`:

```rust
let a = String::from("tic");
let b = String::from("tac");
let c = String::from("toe");
let game = format!("{}-{}-{}", a, b, c);
// a, b, c are all still valid
```

### The UTF-8 Invariant

Here's where `String` gets strict. In Rust, a `String` is **always valid UTF-8**. You cannot create a `String` containing invalid byte sequences in safe Rust. This is enforced by the API — `String::from()`, `push_str()`, `format!()` all accept only valid UTF-8 input.

This has a consequence that surprises people: you cannot index a `String` by position.

```rust
fn main() {
    let hello = String::from("hello");
    let h = hello[0]; // ERROR
}
```

```
error[E0277]: the type `str` cannot be indexed by `{integer}`
 --> src/main.rs:3:13
  |
3 |     let h = hello[0];
  |             ^^^^^^^^ string indices are not supported
  |
  = help: the trait `SliceIndex<str>` is not implemented for `{integer}`
  = note: you can use `.chars().nth()` or `.bytes().nth()`
```

Why? Because UTF-8 characters can be 1 to 4 bytes long. The character "e" is one byte. The character "e" with an accent might be two. An emoji could be four. Index 0 doesn't mean "the first character" — it would mean "the first byte," which might be half a character. Rust refuses to let you do something that would break the UTF-8 invariant.

Use `.chars()` to iterate over characters, or slice carefully with byte ranges when you know the structure:

```rust
fn main() {
    let hello = String::from("hello");
    let first_char = hello.chars().nth(0);
    println!("{:?}", first_char); // Some('h')

    let slice = &hello[0..3]; // byte range — be careful
    println!("{}", slice); // hel
}
```

## HashMap: Unique Keys, Fast Lookups

`HashMap<K, V>` stores key-value pairs. Keys must be unique — inserting a duplicate key overwrites the old value.

### Creating and Inserting

```rust
use std::collections::HashMap;

fn main() {
    let mut player_scores: HashMap<String, i32> = HashMap::new();
    player_scores.insert(String::from("Alice"), 42);
    player_scores.insert(String::from("Bob"), 37);
    player_scores.insert(String::from("Charlie"), 55);

    println!("{:?}", player_scores);
}
```

Unlike `Vec` and `String`, `HashMap` is not in the prelude — you need the `use` statement.

### Accessing Values

Same pattern as `Vec` — direct access can panic, `.get()` returns an `Option`:

```rust
use std::collections::HashMap;

fn main() {
    let mut player_scores = HashMap::new();
    player_scores.insert("Alice", 42);
    player_scores.insert("Bob", 37);

    // .get() returns Option<&V>
    match player_scores.get("Alice") {
        Some(score) => println!("Alice's score: {}", score),
        None => println!("Alice not found"),
    }

    // iterating over key-value pairs
    for (name, score) in &player_scores {
        println!("{}: {}", name, score);
    }
}
```

### The Entry API: Update Patterns

The `entry` API is one of the most useful patterns in Rust. It lets you check if a key exists and insert or modify in one step:

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["hello", "world", "hello", "rust", "hello", "world"];
    let mut word_count: HashMap<&str, i32> = HashMap::new();

    for word in &words {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }

    println!("{:?}", word_count);
}
```

```
{"hello": 3, "rust": 1, "world": 2}
```

`entry()` returns an `Entry` enum — either `Occupied` (key exists) or `Vacant` (key doesn't). `or_insert(0)` inserts the default value only if the key is vacant, then returns a mutable reference to the value either way. This avoids the classic "check then insert" pattern that causes bugs in other languages when the check and insert aren't atomic.

The unique keys invariant means you always know: one key, one value. No accidental duplicates, no ambiguous lookups. If you insert the same key twice, the second value wins.

## What These Collections Share

All three — `Vec`, `String`, `HashMap` — share a common trait: **they own their data**. When they go out of scope, the data is freed. When you push into them, they manage the allocation. You don't call `malloc` or `free`. You don't worry about memory leaks. The ownership system handles it.

Each one also maintains invariants the type system alone doesn't cover:

| Collection | Invariant |
|:--|:--|
| `Vec<T>` | Contiguous memory, valid indices, panics or returns `None` on out-of-bounds |
| `String` | Always valid UTF-8, no indexing by integer, safe character iteration |
| `HashMap<K, V>` | Unique keys, O(1) average lookup, no duplicate entries |

These aren't just implementation details — they're guarantees your code can depend on. A `String` is always valid UTF-8 means you never have to check. A `HashMap` has unique keys means you never have to deduplicate. A `Vec` panics on invalid access means you never silently read garbage.

---

*Collections are where Rust's ownership model meets real-world data. They own their contents, grow as needed, clean up after themselves, and maintain invariants that keep your program correct. With primitives, structs, ownership, borrowing, lifetimes, and collections, you have the core toolkit for writing Rust. The language gets deeper from here — enums, pattern matching, traits, generics — but the foundation is set.*
