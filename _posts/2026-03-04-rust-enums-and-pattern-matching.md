---
title: "Enums and Pattern Matching: The Exhaustiveness Invariant"
date: 2026-03-04 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, enums, pattern-matching, match, option, invariants]
description: "Enums let you define a type by listing its possible variants. match forces you to handle every single one. The compiler guarantees you never forget a case — and Option<T> guarantees you never forget about absence."
---

Structs let you group related data together. But sometimes a value isn't a bundle of fields — it's one of several distinct possibilities. A traffic light is Red, Yellow, or Green. A coin is Heads or Tails. A network request either Succeeded or Failed. These aren't different fields on the same thing. They're different *kinds* of the same thing.

That's what enums are for.

## Defining an Enum

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let heading = Direction::North;
}
```

`Direction` is a type with exactly four possible values. Not five. Not three. Four. The compiler knows every variant, and that knowledge becomes powerful when you need to act on them.

## Variants with Data

Enum variants can carry data — each variant can hold different types and amounts of it:

```rust
enum PlayerAction {
    Move { x: i32, y: i32 },
    Attack(String),
    Heal(u32),
    Idle,
}

fn main() {
    let action = PlayerAction::Move { x: 10, y: -3 };
    let another = PlayerAction::Attack(String::from("fireball"));
    let rest = PlayerAction::Idle;
}
```

`Move` holds a struct-like pair of coordinates. `Attack` holds a `String`. `Heal` holds a `u32`. `Idle` holds nothing. Each variant is a different shape, but they're all the same type: `PlayerAction`.

Try doing this with structs alone and you'd need a struct with optional fields, boolean flags, and a lot of runtime checking to figure out which fields are actually valid. Enums make each possibility explicit and distinct.

## The match Expression

Here's where enums get interesting. Rust provides `match` — a control flow expression that branches based on which variant you have:

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn describe(d: Direction) {
    match d {
        Direction::North => println!("Heading north"),
        Direction::South => println!("Heading south"),
        Direction::East => println!("Heading east"),
        Direction::West => println!("Heading west"),
    }
}

fn main() {
    describe(Direction::North);
    describe(Direction::West);
}
```

```
Heading north
Heading west
```

Each arm of the `match` handles one variant. The syntax is `pattern => expression`. When `d` matches a pattern, the corresponding expression runs.

## Exhaustive Matching

Now let's see what happens when you forget a case. Remove the `West` arm:

```rust
fn describe(d: Direction) {
    match d {
        Direction::North => println!("Heading north"),
        Direction::South => println!("Heading south"),
        Direction::East => println!("Heading east"),
        // forgot West
    }
}
```

The compiler won't let this slide:

```
error[E0004]: non-exhaustive patterns: `Direction::West` not covered
 --> src/main.rs:9:11
  |
9 |     match d {
  |           ^ pattern `Direction::West` not covered
  |
note: `Direction` defined here
 --> src/main.rs:4:5
  |
1 | enum Direction {
  |      ---------
...
4 |     West,
  |     ^^^^ not covered
  = note: the matched value is of type `Direction`
help: ensure that all possible cases are being handled by adding
      a match arm with a wildcard pattern or an explicit pattern
      as shown
  |
12~         Direction::East => println!("Heading east"),
13~         Direction::West => todo!(),
  |
```

This is the core invariant: **`match` is exhaustive**. Every possible variant must be handled. The compiler proves it. You cannot ship code that forgets a case.

This matters most when enums evolve. Add a fifth variant — say `Direction::Up` — and every `match` on `Direction` in your entire codebase that doesn't handle `Up` will fail to compile. The compiler finds every place you need to update. No grep. No "find all references." The type system does it for you.

## Matching with Data

When variants carry data, `match` lets you extract it:

```rust
enum PlayerAction {
    Move { x: i32, y: i32 },
    Attack(String),
    Heal(u32),
    Idle,
}

fn execute(action: PlayerAction) {
    match action {
        PlayerAction::Move { x, y } => {
            println!("Moving to ({}, {})", x, y);
        }
        PlayerAction::Attack(spell) => {
            println!("Casting {}", spell);
        }
        PlayerAction::Heal(amount) => {
            println!("Healing for {} HP", amount);
        }
        PlayerAction::Idle => {
            println!("Doing nothing, as one does");
        }
    }
}

fn main() {
    execute(PlayerAction::Move { x: 5, y: -2 });
    execute(PlayerAction::Attack(String::from("fireball")));
    execute(PlayerAction::Heal(25));
    execute(PlayerAction::Idle);
}
```

```
Moving to (5, -2)
Casting fireball
Healing for 25 HP
Doing nothing, as one does
```

The `match` arm destructures each variant and binds its data to variables. `spell`, `amount`, `x`, `y` — these are all extracted right in the pattern. No casting, no type checking at runtime.

## The Catch-All Pattern

Sometimes you care about a few specific variants and want to handle the rest with a default. The `_` wildcard matches anything:

```rust
fn describe(d: Direction) {
    match d {
        Direction::North => println!("Going up"),
        _ => println!("Going somewhere else"),
    }
}
```

The `_` satisfies the exhaustiveness requirement — it covers every variant you didn't name explicitly. Use it when you genuinely don't need to differentiate between the remaining cases. But be careful: if you add a new variant later, `_` will silently absorb it. You lose the compiler's ability to remind you about unhandled cases. Use `_` deliberately, not lazily.

## Option: Null Doesn't Exist Here

Many languages have `null` — a value that means "nothing" but can pretend to be anything. Null is the source of a staggering number of bugs. Tony Hoare, who invented null references, called it his "billion dollar mistake."

Rust doesn't have null. Instead, it has `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` is an enum with two variants: `Some(T)` holds a value, `None` means absence. It's defined in the standard library and is so fundamental that `Some` and `None` are available without importing anything.

```rust
fn main() {
    let got_the_thing: Option<i32> = Some(42);
    let nope: Option<i32> = None;
}
```

The key insight: `Option<i32>` and `i32` are different types. You cannot use an `Option<i32>` where an `i32` is expected. The compiler forces you to handle the `None` case before you can access the value inside `Some`.

```rust
fn main() {
    let maybe_number: Option<i32> = Some(10);
    let actual_number: i32 = 5;

    // let sum = maybe_number + actual_number; // ERROR
}
```

```
error[E0277]: cannot add `i32` to `Option<i32>`
 --> src/main.rs:5:32
  |
5 |     let sum = maybe_number + actual_number;
  |                            ^ no implementation for `Option<i32> + i32`
```

You must explicitly extract the value first. The compiler makes absence visible and forces you to deal with it. No null pointer exceptions. No "undefined is not a function." If a value might not exist, the type says so.

## Extracting Values from Option

Use `match` to handle both cases:

```rust
fn double_or_zero(x: Option<i32>) -> i32 {
    match x {
        Some(val) => val * 2,
        None => 0,
    }
}

fn main() {
    println!("{}", double_or_zero(Some(21))); // 42
    println!("{}", double_or_zero(None));      // 0
}
```

Every code path is covered. The `Some` case extracts the value. The `None` case provides a fallback. The compiler checked both.

## if let: The Shortcut

When you only care about one variant and want to ignore the rest, `match` can feel verbose. `if let` is the concise alternative:

```rust
fn main() {
    let favorite_number: Option<i32> = Some(7);

    if let Some(num) = favorite_number {
        println!("My favorite number is {}", num);
    }
}
```

```
My favorite number is 7
```

`if let` is syntactic sugar for a `match` with one arm and a wildcard. It's great when you want to do something for `Some` and nothing for `None`. You can pair it with `else` too:

```rust
fn main() {
    let mystery_box: Option<&str> = None;

    if let Some(contents) = mystery_box {
        println!("Found: {}", contents);
    } else {
        println!("The box is empty");
    }
}
```

```
The box is empty
```

Use `if let` for convenience. Use `match` when you need to handle every variant explicitly.

## The Guarantees

Enums and pattern matching encode a set of invariants that the compiler enforces at every use site:

| Mechanism | What It Guarantees |
|:--|:--|
| Enum definition | A value is exactly one of the listed variants — nothing else |
| `match` exhaustiveness | Every possible variant is handled — no forgotten cases |
| `Option<T>` instead of null | Absence is explicit in the type — you cannot ignore it |
| Pattern destructuring | Data extraction is type-safe — no invalid casts |

These aren't runtime checks. They're compile-time proofs. When you add a variant, the compiler finds every place that needs updating. When a value might be absent, the type system makes you deal with it. The guarantees hold across your entire codebase, not just in the tests you remembered to write.

---

*Enums define what's possible. `match` ensures you handle all of it. `Option` makes absence a first-class concept instead of a hidden trap. Next up: error handling with `Result<T, E>` — Rust's way of making sure you never silently ignore a failure.*
