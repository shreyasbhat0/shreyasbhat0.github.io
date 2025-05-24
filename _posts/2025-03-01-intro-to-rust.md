---
title: "Intro to Rust: A Language Built on Invariants"
date: 2025-03-01 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, introduction, memory-safety, invariants]
description: "Rust isn't just another systems language. It's a language that encodes invariants — memory safety, thread safety, type correctness — directly into the compiler. Here's where it starts."
---

Most languages ask you to remember the rules. Rust *enforces* them.

The race to replace C has been running for decades, and Rust is the clear winner — not because it's faster or has better syntax, but because it takes properties that C developers had to uphold through discipline and makes them **invariants** the compiler guarantees. No dangling pointers. No data races. No use-after-free. Not through testing, not through code review, but through the type system itself.

## What is Rust?

Rust was designed by Graydon Hoare while working at Mozilla Research in 2006. It's a multi-paradigm, strongly typed, memory-efficient language designed for safety and performance.

But that's the marketing version. Here's what matters: Rust guarantees **memory safety** and **thread safety** at compile time. These aren't aspirational goals — they're invariants. If your code compiles, these properties hold. If they don't hold, your code doesn't compile.

This is a fundamentally different contract from what most languages offer.

## Installation

**Linux and macOS:**

```bash
curl --proto '=https' --tlsv1.3 https://sh.rustup.rs -sSf | sh
```

This downloads and installs `rustup`, which manages your Rust toolchain. If you encounter linker errors, install a C compiler — some Rust packages depend on C code.

On macOS:

```bash
xcode-select --install
```

On Linux:

```bash
sudo apt install build-essential
```

**Windows:**

Refer to the [official Rust setup guide for Windows](https://learn.microsoft.com/en-us/windows/dev-environment/rust/setup).

## Your First Program

Create a project directory and a source file:

```bash
mkdir projects
cd projects
```

Create a file called `first_program.rs`:

```rust
fn main() {
    println!("Hello, World!");
}
```

Compile it:

```bash
rustc first_program.rs
```

This produces an executable binary. Run it:

```bash
./first_program
```

If `Hello, World!` appears in your terminal, you've written and executed your first Rust program.

## What Just Happened, in Terms of Invariants

Even in this trivial program, several invariants are being enforced:

- **Type safety** — `println!` is a macro that checks format strings at compile time. Pass the wrong type and the program won't compile.
- **Memory safety** — `"Hello, World!"` is a string literal with a static lifetime. There's no allocation, no deallocation, no possibility of a dangling pointer.
- **Entry point contract** — `fn main()` is the program's entry point. The compiler enforces its existence and signature.

These seem obvious in a "Hello, World!" program. They become critical when your program is 50,000 lines of concurrent systems code. The invariant is the same at both scales.

---

*This is the first post in a series on Rust fundamentals, viewed through the lens of invariants — the properties that must hold true in every system.*
