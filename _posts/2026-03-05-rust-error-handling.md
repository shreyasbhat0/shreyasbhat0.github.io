---
title: "Error Handling: No Silent Failures"
date: 2026-03-05 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, error-handling, result, option, panic, invariants]
description: "Rust splits errors into two categories: unrecoverable (panic) and recoverable (Result). The compiler forces you to handle the recoverable ones. You cannot ignore a Result — the type system won't let you."
---

Most languages let you ignore errors. A function returns an error code and you don't check it. An exception gets thrown and you don't catch it. The program keeps running, silently wrong, until something breaks downstream and you spend three hours figuring out that the real failure happened 200 lines ago.

Rust doesn't let you do this. Errors are values, and the type system tracks them. If a function can fail, its return type says so. If you don't handle the failure, the compiler tells you.

## Two Kinds of Errors

Rust divides errors into two categories:

- **Unrecoverable** — something has gone so wrong that the program cannot continue. A bug. A violated assumption. Use `panic!`.
- **Recoverable** — something went wrong, but the caller can decide what to do about it. A missing file. A network timeout. An invalid input. Use `Result<T, E>`.

This distinction is a design decision you make at every function boundary: *can the caller do something useful with this failure?*

## panic! — The Emergency Exit

`panic!` crashes the program immediately, printing an error message and unwinding the stack:

```rust
fn main() {
    panic!("something went terribly wrong");
}
```

```
thread 'main' panicked at 'something went terribly wrong', src/main.rs:2:5
```

Use `panic!` when the program has hit a state that should be impossible — an invariant has been violated, and continuing would only make things worse. Index out of bounds? Panic. A `None` where you are absolutely certain there should be a `Some`? Panic. A logic error that means your assumptions about the world are wrong? Panic.

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    println!("{}", numbers[99]); // panics: index out of bounds
}
```

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index
is 99', src/main.rs:3:20
```

Panics are not for expected failures. A file that might not exist is not a reason to panic — it's a reason to return a `Result`.

## Result<T, E>

`Result` is an enum with two variants:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Ok(T)` holds the success value. `Err(E)` holds the error value. Like `Option`, it's in the standard library and available everywhere without importing.

Consider reading a file:

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("hello.txt");
    println!("{:?}", content);
}
```

If `hello.txt` exists, `content` is `Ok("file contents here...")`. If it doesn't, `content` is `Err(Os { code: 2, kind: NotFound, message: "No such file or directory" })`.

The function didn't crash. It didn't return null. It returned a value that explicitly says "this failed, and here's why." Now you decide what to do.

## Handling Result with match

The familiar pattern:

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("hello.txt");

    match content {
        Ok(text) => println!("File contents: {}", text),
        Err(error) => println!("Could not read file: {}", error),
    }
}
```

Both paths are handled. The compiler checked it. If you forget the `Err` arm, you get the same exhaustiveness error as with any other enum — `match` forces you to handle every variant.

## What Happens When You Ignore a Result

Let's try calling a function that returns `Result` and not doing anything with it:

```rust
use std::fs;

fn main() {
    fs::read_to_string("hello.txt");
}
```

The compiler won't let this pass quietly:

```
warning: unused `Result` that must be used
 --> src/main.rs:4:5
  |
4 |     fs::read_to_string("hello.txt");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
  = note: `#[warn(unused_must_use)]` on by default
```

`Result` is marked with `#[must_use]`. The compiler warns you if you discard it. You can't accidentally ignore an error — you have to *deliberately* decide what to do with it. This is the core invariant: **every error is either handled or explicitly acknowledged**.

## unwrap and expect

Sometimes you're sure a function won't fail, or you're prototyping and don't want to write full error handling yet. `unwrap` extracts the `Ok` value, panicking if it's `Err`:

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("hello.txt").unwrap();
    println!("{}", content);
}
```

If the file exists, this works fine. If it doesn't:

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value:
Os { code: 2, kind: NotFound, message: "No such file or directory" }',
src/main.rs:4:52
```

`expect` does the same thing but lets you provide a custom message:

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("config.toml")
        .expect("config.toml must exist in the project root");
    println!("{}", content);
}
```

```
thread 'main' panicked at 'config.toml must exist in the project root:
Os { code: 2, kind: NotFound, message: "No such file or directory" }',
src/main.rs:4:10
```

`expect` is better than `unwrap` because the panic message tells you *why* the value was expected to be present. Use `expect` over `unwrap` in real code. But both are escape hatches — they convert a recoverable error into a panic. Use them when failure genuinely means "something is broken," not when failure is a normal possibility.

## The ? Operator

In real programs, you often want to propagate errors up to the caller rather than handling them on the spot. The `?` operator does exactly this:

```rust
use std::fs;
use std::io;

fn read_config() -> Result<String, io::Error> {
    let content = fs::read_to_string("config.toml")?;
    Ok(content)
}

fn main() {
    match read_config() {
        Ok(config) => println!("Config: {}", config),
        Err(e) => println!("Failed to read config: {}", e),
    }
}
```

The `?` after `fs::read_to_string("config.toml")` says: "If this is `Ok`, unwrap the value and continue. If this is `Err`, return the error from the current function immediately."

Without `?`, you'd write this:

```rust
fn read_config() -> Result<String, io::Error> {
    let content = match fs::read_to_string("config.toml") {
        Ok(text) => text,
        Err(e) => return Err(e),
    };
    Ok(content)
}
```

The `?` operator is just shorthand for that `match`. It doesn't hide errors — it propagates them. The function's return type still says `Result`, so every caller knows this function can fail. The error doesn't disappear; it moves up the call stack explicitly.

You can chain `?` for multiple fallible operations:

```rust
use std::fs;
use std::io;

fn read_and_count() -> Result<usize, io::Error> {
    let content = fs::read_to_string("data.txt")?;
    let trimmed = content.trim();
    let line_count = trimmed.lines().count();
    Ok(line_count)
}
```

If `read_to_string` fails, the function returns the error immediately. If it succeeds, execution continues. Clean, readable, and every error path is visible in the return type.

One important restriction: `?` can only be used in functions that return `Result` (or `Option`). Try using it in a function that returns nothing:

```rust
fn main() {
    let content = fs::read_to_string("hello.txt")?;
}
```

```
error[E0277]: the `?` operator can only be used in a function that returns
`Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:52
  |
3 | fn main() {
  | --------- this function should return `Result` or `Option` to accept `?`
4 |     let content = fs::read_to_string("hello.txt")?;
  |                                                    ^ cannot use the `?`
  |                                                      operator in a
  |                                                      function that
  |                                                      returns `()`
```

The compiler enforces this: you can't propagate an error out of a function that doesn't declare it can fail. The return type is the contract, and `?` respects it.

## When to panic vs When to Return Result

The decision is straightforward:

| Situation | Use |
|:--|:--|
| Bug in your code — invariant violated | `panic!` |
| Prototype or example code | `unwrap` / `expect` |
| Expected failure the caller should handle | `Result<T, E>` |
| Configuration that *must* exist | `expect` with a clear message |
| User input, file I/O, network calls | `Result<T, E>` |

If the error represents something *your code* did wrong, panic. If the error represents something *the world* did wrong (bad input, missing files, network failures), return `Result` and let the caller decide.

## Custom Error Types

For libraries or larger programs, you'll want your own error type instead of passing around raw `io::Error` or `String`:

```rust
#[derive(Debug)]
enum AppError {
    NotFound(String),
    InvalidInput(String),
    IoError(std::io::Error),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::NotFound(path) => write!(f, "not found: {}", path),
            AppError::InvalidInput(msg) => write!(f, "invalid input: {}", msg),
            AppError::IoError(e) => write!(f, "io error: {}", e),
        }
    }
}

impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self {
        AppError::IoError(e)
    }
}
```

With the `From` implementation, the `?` operator automatically converts `io::Error` into `AppError`. Your functions return `Result<T, AppError>`, and callers get a single error type to match on. The exhaustiveness guarantee applies here too — add a new variant to `AppError`, and every `match` on it must be updated.

## The Error Handling Invariants

| Mechanism | What It Guarantees |
|:--|:--|
| `Result<T, E>` | Fallible operations declare failure in their type |
| `#[must_use]` on Result | You cannot silently ignore an error |
| `match` on Result | Both `Ok` and `Err` must be handled |
| `?` operator | Errors are propagated explicitly, never hidden |
| `panic!` | Unrecoverable errors stop execution immediately |
| Custom error types | Exhaustive matching on all error variants |

The thread running through all of this: **errors are visible**. They live in the type system, not in exception tables or error codes you might forget to check. A function that can fail says so in its signature. A caller that ignores the failure gets a compiler warning. An error that propagates up the call stack does so through `?`, leaving a visible trail in every function's return type.

No silent failures. No "it worked on my machine." If the program compiles, every error path is accounted for.

---

*`Result` and `Option` are two sides of the same design: make the unhappy path explicit. `Option` says "this might be absent." `Result` says "this might fail." In both cases, the compiler ensures you deal with it. That's the pattern — Rust doesn't trust you to remember. It makes forgetting a compile error.*
