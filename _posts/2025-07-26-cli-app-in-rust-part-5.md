---
title: "Building a CLI App in Rust, Part 5: Error Handling as Invariant Enforcement"
date: 2025-07-26 10:00:00 +0530
categories: [Rust, CLI]
tags: [rust, clap, cli, invariants, error-handling]
description: "Every .unwrap() is an invariant you haven't enforced. Proper error handling transforms silent crashes into explicit contracts about what can go wrong and how the program responds."
---

In the previous four posts, we built Cryptifer from the ground up — CLI parsing, key generation, encryption, decryption. The round-trip invariant holds: encrypting and decrypting with the same key restores the original plaintext.

But there is an invariant we have been violating throughout the entire series: **the program must never crash without explanation**.

Look at our code. It is littered with `.unwrap()` calls. Every one of them is a hidden `panic!` — an uncontrolled termination that prints a stack trace instead of a useful error message. When a user provides a nonexistent file path, they should see "file not found: /path/to/file", not a Rust panic dump pointing at line 47 of `generate.rs`.

This post is about fixing that. In Rust, error handling is not just good practice — it is invariant enforcement. The type system gives us `Result<T, E>`, which encodes a fundamental invariant: **every operation that can fail must declare that it can fail, and every caller must handle the failure**.

## Errors in Rust

Rust groups errors into two categories:

- **Recoverable errors** — represented by `Result<T, E>`. The operation failed, but the program can respond meaningfully (retry, fall back, report to the user).
- **Unrecoverable errors** — triggered by `panic!`. The program is in a state so broken that continuing would cause worse problems (out-of-memory, violated safety invariants).

The invariant is clear: **only truly unrecoverable situations should panic. Everything else should return a `Result`**.

In our current code, we use `.unwrap()` for everything — file I/O, JSON parsing, cipher creation, base64 decoding. None of these are unrecoverable. A missing file is not a reason to crash; it is a reason to tell the user which file is missing and exit cleanly.

## The Problem with `.unwrap()`

Consider this line from `generate.rs`:

```rust
let mut file = File::create(out_path).unwrap();
```

If the output path is in a directory that does not exist, `File::create` returns `Err(io::Error)`, and `.unwrap()` panics with a message like:

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Os { code: 2, kind: NotFound, message: "No such file or directory" }'
```

This message technically contains the information — but it is buried in Rust internals. The user has to parse debug formatting, understand what `Result::unwrap()` means, and extract the actual problem. Compare that to:

```
Error: cannot create output file '/nonexistent/path/Keystore.secret': No such file or directory
```

The second message upholds an invariant about the relationship between the program and the user: **every error message must be actionable**. The user knows exactly what went wrong and what to fix.

## Refactoring with `Result`

The refactoring pattern is consistent across the codebase:

1. Change functions that can fail to return `Result<T, E>` instead of panicking.
2. Use the `?` operator to propagate errors up to the caller.
3. Handle errors at the top level (`main`) with user-friendly messages.

For example, the `generate` function changes from:

```rust
pub fn new(out_path: String) {
    // ... generate key and IV ...
    let mut file = File::create(out_path).unwrap();
    file.write_all(&secret).unwrap();
    println!("Secret Generated Successfully")
}
```

to a version that returns `Result`:

```rust
pub fn new(out_path: String) -> Result<(), Box<dyn std::error::Error>> {
    // ... generate key and IV ...
    let mut file = File::create(&out_path)
        .map_err(|e| format!("cannot create output file '{}': {}", out_path, e))?;
    file.write_all(&secret)
        .map_err(|e| format!("failed to write secret to '{}': {}", out_path, e))?;
    println!("Secret Generated Successfully");
    Ok(())
}
```

The `?` operator is the key. It says: "if this operation failed, return the error to the caller immediately." The invariant — "this function either succeeds completely or reports exactly why it failed" — is encoded in the return type. The caller cannot ignore a failure because `Result` forces them to handle it.

The same pattern applies to `encrypt` and `decrypt`:

- `read(file_path)` can fail if the file does not exist.
- `serde_json::from_slice` can fail if the secret file is corrupted.
- `AES::new_from_slices` can fail if the key or IV is the wrong length.
- `cipher.encrypt` / `cipher.decrypt` can fail if the data is malformed.
- `base64::decode` can fail if the ciphertext contains invalid characters.
- `str::from_utf8` can fail if the decrypted bytes are not valid UTF-8.

Every one of these failure points is a potential invariant violation. Proper error handling does not prevent the violation — it ensures the violation is **detected, reported, and handled gracefully** rather than causing an uncontrolled crash.

## Error Handling in `main`

The top-level `main` function becomes the error boundary — the place where all propagated errors are caught and presented to the user:

```rust
fn main() {
    let cryptifer = Cryptifer::parse();

    let result = match cryptifer.command {
        commands::Commands::Generate { output_path } => {
            Secret::new(output_path)
        }
        commands::Commands::Encrypt { file_path, key_path } => {
            encrypt(file_path, key_path)
        }
        commands::Commands::Decrypt { encrypted_file, key_path } => {
            decrypt(encrypted_file, key_path)
        }
    };

    if let Err(e) = result {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}
```

This establishes a clean invariant for the program's exit behavior:
- **Exit code 0**: the operation completed successfully.
- **Exit code 1**: the operation failed, and the error is printed to stderr.

No panics. No stack traces. The program either works or explains why it did not.

## The Invariant Hierarchy

Looking at the complete Cryptifer application, we can see a hierarchy of invariants, each depending on the ones below it:

| Layer | Invariant | Enforced By |
|:--|:--|:--|
| CLI Parsing | All required arguments are present and well-typed | Clap (compile-time + runtime) |
| Key Generation | Key and IV are 16 bytes, random, base64-encoded | Fixed-size arrays, CSPRNG, serde |
| Encryption | Ciphertext is produced by correctly-configured AES-CBC | Type alias, `new_from_slices` validation |
| Decryption | Round-trip: `decrypt(encrypt(P, K), K) == P` | Symmetric cipher construction, matching encoding |
| Error Handling | Every failure is reported, never silent | `Result<T, E>`, `?` operator, exhaustive matching |

The bottom layer — error handling — is not the least important. It is the *most* important, because it is what makes all the other invariants observable. An invariant that fails silently is worse than no invariant at all. It gives you false confidence while the system drifts into an inconsistent state.

## The Complete Application

The refactored code with proper error handling is available at [Cryptifer](https://github.com/shreyasbhat0/cryptifier).

## Wrapping Up

This series covered more than building a CLI app. Each post was about identifying and enforcing an invariant:

1. **Part 1**: The input shape invariant — valid commands with valid arguments.
2. **Part 2**: The key generation invariant — correct length, sufficient randomness, lossless encoding.
3. **Part 3**: The confidentiality invariant — ciphertext reveals nothing without the key.
4. **Part 4**: The round-trip invariant — decrypt undoes encrypt, exactly.
5. **Part 5**: The error handling invariant — every failure is explicit, every success is confirmed.

Rust is uniquely suited to this style of thinking. The type system, the borrow checker, exhaustive pattern matching, and the `Result` type all exist to make invariant violations either impossible (at compile time) or immediately visible (at runtime). When you write Rust, you are not just writing logic — you are declaring what must always be true, and the compiler holds you to it.

The tools change. The invariants do not.
