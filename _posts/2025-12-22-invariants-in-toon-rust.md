---
title: "Invariants in the Wild: Implementing TOON's Rust Parser"
date: 2025-12-22 18:00:00 +0530
categories: [Rust, Parsing]
tags: [rust, toon, invariants, parsing, llm]
description: "Invariants aren't just for distributed systems. They're in every parser, every format, every function boundary. Here's where I found them while implementing TOON in Rust."
---

When people hear "invariant," they think distributed systems. Consensus protocols. Database isolation levels. Big, serious infrastructure.

But invariants are everywhere. They're in your CSV parser. In your config loader. In the format you chose for serializing data. Every piece of software that transforms input into output has invariants — properties that must hold true before, during, and after the transformation. Most of the time, we just don't name them.

This post is about finding and naming them — in a place you might not expect: a data format parser.

## The Context

[TOON](https://github.com/toon-format/spec) (Token-Oriented Object Notation) is a compact, line-oriented format designed by [Johann Schopplich](https://github.com/johannschopplich). It encodes the JSON data model with minimal quoting and explicit structure. The motivation: **JSON wastes tokens when fed to LLMs**, and tokens cost money.

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

Fewer tokens. Same data. I wrote [toon-rust](https://github.com/toon-format/toon-rust), the Rust implementation of this spec. What surprised me wasn't the parsing logic — it was how many invariants I had to discover, name, and enforce along the way.

## Invariant #1: Length Markers Never Lie

TOON arrays declare their length upfront: `[3]` means exactly three rows follow. This seems like metadata. It isn't. It's a **contract between the encoder and the decoder**.

```
[3] name,age
Alice,30
Bob,25
```

Header says 3 rows. Only 2 exist. What should happen?

A lenient parser might silently accept this — return what it has, ignore the discrepancy. This is the kind of decision that seems pragmatic and is actually dangerous. Downstream code allocated space for 3 items. An LLM prompt was constructed expecting 3 examples. A batch job assumes the count is accurate. One silent lie propagates into many silent failures.

In toon-rust, this is a hard error:

```rust
ToonError::LengthMismatch {
    expected: 3,
    found: 2,
    context: Some(/* source location, suggestion */),
}
```

The invariant: **declared length ≡ actual row count**. Not approximately. Not "best effort." Exactly equal. Every decode path checks this, and there is no flag to disable it. The format promised something. The parser holds it to that promise.

This pattern shows up everywhere once you start looking. HTTP `Content-Length` headers. Database schema column counts. Array bounds in every language with bounds checking. The invariant is always the same: *if you declare a size, the actual size must match*.

## Invariant #2: Delimiters Are Scoped, Not Global

TOON supports three delimiters: comma, tab, and pipe. The obvious design would be to pick one delimiter globally. TOON doesn't do this — a delimiter is scoped to a specific array context. Nested structures can use different delimiters at different levels.

This creates a subtle invariant: **the active delimiter at any parse point is determined by the nearest enclosing array header**. Get this wrong by even one level, and you split fields at the wrong boundary. The data parses successfully, produces valid-looking output, and is completely wrong. No error. No crash. Just wrong data.

This is the most insidious class of invariant violation — the kind where the system keeps running. In distributed systems, we call this "silent data corruption." In parsing, it's the same thing: the bytes look fine, the structure looks fine, the values are garbage.

In Rust, the scoping maps to a stack. Enter an array, push its delimiter. Leave, pop. The discipline is manual, but the structured error types mean a scoping mistake surfaces as a `ParseError` rather than a silent misparse. The key insight: **scope is an invariant**. Every system that has nested contexts — variable scoping in compilers, transaction isolation in databases, CSS specificity in browsers — has this same invariant hiding inside it.

## Invariant #3: Round-Trip Fidelity

The fundamental invariant of any serialization format: `decode(encode(value)) ≡ value`.

This sounds like it should be automatic. It never is. Consider:

- Is the number `1.0` the same as `1`? In JSON they're both valid representations of the same value. In a typed system, one is a float and the other is an integer.
- The string `true` — is it a boolean or a string that happens to contain the word "true"? When do you quote it?
- An empty array `[]` and an empty object `{}` — both are empty containers, but they are not the same thing. Can your format distinguish them?

Every one of these edge cases is an invariant violation waiting to happen. The data goes in as one type and comes out as another. The system doesn't crash — it just changes the meaning of your data underneath you.

TOON's spec addresses this with **strict mode** — a validation layer that rejects anything ambiguous. In toon-rust, this becomes two separate functions:

```rust
pub fn decode_strict(input: &str) -> ToonResult<Value> { ... }
pub fn decode_default(input: &str) -> ToonResult<Value> { ... }
```

Same input type. Different invariants. The strict decoder promises: *if I return Ok, the input is fully spec-compliant and unambiguous*. The default decoder promises: *if I return Ok, the data is semantically correct, even if the formatting is loose*.

This is a pattern worth stealing. Instead of one function that tries to be everything, have two that each **declare which invariant they uphold**. The caller chooses the guarantee they need. You see the same idea in Rust's `str::parse` vs `str::parse_lossy`, in PostgreSQL's `STRICT` vs `PERMISSIVE` modes, in HTTP's content negotiation. The principle is universal: *make the strength of your guarantee explicit*.

## Invariant #4: Every Rejection Has a Reason

A parser that says "parse failed at byte 847" has technically upheld correctness. But it has violated a different invariant — one about the relationship between the system and the human operating it.

The invariant: **every rejection must carry enough context to understand and fix the problem**.

```rust
pub struct ErrorContext {
    pub source_line: String,
    pub preceding_lines: Vec<String>,
    pub following_lines: Vec<String>,
    pub suggestion: Option<String>,
    pub indicator: Option<String>,
}
```

Every `ParseError` and `LengthMismatch` in toon-rust carries an `ErrorContext`. The column indicator (`^`) points to the exact character. Preceding and following lines show the neighborhood. The suggestion field offers a fix.

This isn't polish — it's an invariant about error quality. The Rust compiler itself is the gold standard here: it doesn't just reject invalid code, it explains *why* and *what you probably meant*. That design choice is what makes Rust learnable despite its complexity. The same principle applies to any system that rejects user input: reject clearly, or don't reject at all.

## What Rust Enforces at Compile Time

Some invariants don't need runtime checks because the type system makes violations impossible:

- **`ToonResult<T>`** — every fallible operation returns a `Result`. You cannot forget to handle a `LengthMismatch`. The compiler won't let you.
- **Exhaustive matching** — add a new variant to `ToonError`, and every `match` that doesn't handle it fails to compile. The invariant "all error cases are handled" is enforced automatically.
- **Ownership and borrowing** — the parser can't accidentally alias mutable state. If two decode paths need to share a buffer, the borrow checker forces you to be explicit. No data races. No use-after-free. Not because of discipline, but because of the type system.

This is the deepest lesson from implementing toon-rust: **the best invariants are the ones you don't have to check**. When the type system makes a violation unrepresentable, you've moved the invariant from "something we test for" to "something that cannot happen." That's not an incremental improvement. That's a category change.

## Invariants Are Everywhere

A parser feels like a small, self-contained problem. But every invariant I found in toon-rust has a direct analog in larger systems:

| Parser Invariant | System Analog |
|:--|:--|
| Length markers match actual data | Schema validation, Content-Length headers |
| Delimiter scoping | Variable scoping, transaction isolation |
| Round-trip fidelity | Serialization guarantees, database ACID |
| Errors carry context | Observability, structured logging |
| Type-level enforcement | Static analysis, formal verification |

The pattern is the same at every scale. The only difference is the blast radius when an invariant breaks — a parser produces wrong data, a distributed system loses consistency, a financial system loses money. The discipline of finding and naming invariants doesn't change.

You don't need to be building a consensus protocol to think in invariants. You just need to ask, at every function boundary: **what must always be true here?**

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
