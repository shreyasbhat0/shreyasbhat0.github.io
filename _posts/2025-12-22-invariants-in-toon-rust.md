---
title: "Invariants in the Wild: Implementing TOON's Rust Parser"
date: 2025-12-22 18:00:00 +0530
categories: [Rust, Parsing]
tags: [rust, toon, invariants, parsing, llm]
description: "How enforcing invariants at the type level shaped the Rust implementation of TOON — a token-efficient data format for LLMs."
---

Every parser is a machine for enforcing invariants. You take unstructured bytes, and you either produce a valid structure — or you reject. There is no middle ground. A parser that "mostly works" is a parser that silently corrupts data.

This is the story of implementing [TOON](https://github.com/toon-format/spec) (Token-Oriented Object Notation) in Rust — not as a tutorial on the format, but as a study in the invariants that hold a parser together.

## What is TOON?

TOON is a compact, line-oriented format that encodes the JSON data model with minimal quoting and explicit structure. It was designed by [Johann Schopplich](https://github.com/johannschopplich) for a specific problem: **JSON wastes tokens when fed to LLMs**.

Consider a simple dataset in JSON:

```json
[
  {"name": "Alice", "age": 30, "role": "engineer"},
  {"name": "Bob", "age": 25, "role": "designer"}
]
```

The same data in TOON:

```
[2] name,age,role
Alice,30,engineer
Bob,25,designer
```

Fewer tokens. Same data. The invariant is clear: **encode(data) → TOON → decode(TOON) ≡ data**. Round-trip fidelity is non-negotiable.

I wrote [toon-rust](https://github.com/toon-format/toon-rust), the Rust implementation of this spec. What follows are the invariants I had to enforce — and how Rust made some of them free.

## Invariant #1: Length Markers Never Lie

TOON arrays declare their length upfront: `[3]` means exactly three rows follow. This isn't optional metadata — it's a **contract**.

```
[3] name,age
Alice,30
Bob,25
```

This is a `LengthMismatch`. The header promises 3 rows. Only 2 exist. In a lenient parser, you might silently accept this. In toon-rust, it's an error:

```rust
ToonError::LengthMismatch {
    expected: 3,
    found: 2,
    context: Some(/* source location, suggestion */),
}
```

The invariant: **declared length ≡ actual row count**. Every decode path checks this. There is no flag to skip it — because skipping it means the data model is lying about itself.

## Invariant #2: Delimiters Are Scoped, Not Global

TOON supports three delimiters: comma, tab, and pipe. But a delimiter isn't a global setting — it's scoped to a specific array context. Nested structures can use different delimiters.

This creates an invariant the parser must maintain: **the active delimiter at any point is determined by the nearest enclosing array header**. Get this wrong and you split fields at the wrong boundary. The data looks valid but means something entirely different.

In Rust, this naturally maps to a stack:

```rust
struct Delimiter {
    kind: DelimiterKind,  // Comma, Tab, Pipe
}
```

Each time we enter an array, we push its delimiter. Each time we leave, we pop. The type system doesn't enforce the stack discipline for us here — but the structured error types mean a wrong delimiter immediately surfaces as a `ParseError`, not a silent misparse.

## Invariant #3: Round-Trip Fidelity

The hardest invariant to maintain: `decode(encode(value)) ≡ value`.

This sounds obvious. It isn't. Consider edge cases:

- Numbers: is `1.0` the same as `1`? In JSON, yes. In a type-aware system, maybe not.
- Strings that look like keywords: `true`, `false`, `null` — when do you quote them?
- Empty containers: `[]` vs `{}` vs absence.

The spec defines **strict mode** for exactly this reason — a validation layer that rejects anything ambiguous. In toon-rust, strict decoding is a separate code path:

```rust
pub fn decode_strict(input: &str) -> ToonResult<Value> { ... }
pub fn decode_default(input: &str) -> ToonResult<Value> { ... }
```

Two functions. Same input type. Different invariants. The strict decoder promises: *if I return Ok, the input is spec-compliant*. The default decoder promises: *if I return Ok, the data is semantically correct, even if the formatting is loose*.

Both are valid. The key is that each function **declares which invariant it upholds**.

## Invariant #4: Errors Carry Their Proof

A parser error that says "parse failed" is useless. An error that says "line 7, column 3: expected 3 fields but found 2, here's the line, here's what you probably meant" — that's an invariant about error quality.

```rust
pub struct ErrorContext {
    pub source_line: String,
    pub preceding_lines: Vec<String>,
    pub following_lines: Vec<String>,
    pub suggestion: Option<String>,
    pub indicator: Option<String>,
}
```

Every `ParseError` and `LengthMismatch` can carry an `ErrorContext`. This isn't optional niceness — it's a design invariant of the crate. When the parser rejects your input, it must **prove why**. The column indicator (`^`) points to the exact character. The suggestion tells you what to fix.

This is the difference between a parser that says "no" and a parser that says "no, and here's what you probably meant." Both uphold correctness. One is also humane.

## What Rust Gives You for Free

Not all invariants need to be enforced at runtime. Rust's type system provides some as compile-time guarantees:

- **`ToonResult<T>`** — every operation that can fail returns a `Result`. You cannot forget to handle an error. The caller *must* decide what to do with a `LengthMismatch`.
- **`#[derive(Clone, PartialEq)]` on errors** — errors are values you can compare, test, and match on. This means invariants about error behavior are testable.
- **Exhaustive pattern matching** — add a new error variant, and every `match` in the codebase that doesn't handle it fails to compile.
- **Ownership** — the parser can't accidentally alias mutable state. If two decode paths share a buffer, the borrow checker forces you to be explicit about it.

These aren't features of toon-rust. They're features of Rust. But choosing Rust for a parser *is* choosing to encode your invariants at the type level wherever possible.

## The Meta-Invariant

Every format implementation has one invariant that governs all others:

> **The implementation must be a faithful model of the specification.**

Not a superset. Not a "practical" subset. A faithful model. When the TOON spec says arrays declare their length, the Rust implementation enforces it. When the spec says strict mode rejects ambiguous input, the Rust implementation rejects it.

This is harder than it sounds, because specs are written in English and implementations are written in Rust. The translation is where bugs live. The discipline of thinking in invariants — asking "what must always be true here?" at every function boundary — is what keeps the translation honest.

---

## Try It

```bash
cargo add toon-format
```

```rust
use toon_format::{encode, decode};
use serde_json::json;

let data = json!([
    {"name": "Alice", "age": 30},
    {"name": "Bob", "age": 25}
]);

let toon = encode(&data)?;
let roundtrip = decode(&toon)?;

assert_eq!(data, roundtrip); // The invariant holds.
```

The code is at [toon-format/toon-rust](https://github.com/toon-format/toon-rust). The spec is at [toon-format/spec](https://github.com/toon-format/spec). Contributions welcome.

---

*Next: invariants in distributed consensus — what Raft actually guarantees and what it doesn't.*
