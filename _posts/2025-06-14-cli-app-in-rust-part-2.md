---
title: "Building a CLI App in Rust, Part 2: Key Generation and the Invariant of Sufficient Randomness"
date: 2025-06-14 10:00:00 +0530
categories: [Rust, CLI]
tags: [rust, clap, cli, invariants, cryptography, key-generation]
description: "A cryptographic key that is too short, predictable, or improperly encoded is worse than no key at all. The generate command must uphold a strict invariant: the output is always a correctly-sized, random, properly-encoded secret."
---

In the previous post, we set up the Cryptifer CLI skeleton with Clap, establishing invariants about input shape — every subcommand requires its arguments, and the parser rejects anything that does not conform.

Now we implement the first command: `generate`. This command produces a secret containing a cryptographic key and an IV (initial value), then writes them to an output file.

The invariant here is precise: **the generated key and IV must each be exactly 16 bytes of cryptographically random data, base64-encoded, and written atomically to the output path**. A key that is too short breaks encryption. A key that is not random is predictable. A partially-written file corrupts the secret. Every part of this invariant matters.

## Implementing the Generate Command

Create a new file `generate.rs` in the `src` directory. First, add the `rand` dependency:

```
cargo add rand
```

We also need `serde` for serialization. Add these to `Cargo.toml`:

```
serde_json = "1.0.85"
serde = "1.0.85"
serde_derive = "1.0.85"
```

More on [serde](https://crates.io/crates/serde).

Import the new dependencies and the `generate` module in `main.rs`:

```rust
use clap::{Parser, Subcommand};
use generate::Secret;
use rand::*;
mod commands;
use commands::Cryptifer;
mod generate;
use serde_derive::{Deserialize, Serialize};
use serde_json::to_vec;
use std::fs::File;
use std::io::Write;
```

Now define the `Secret` struct in `generate.rs`. This struct holds two fields — `key` and `inital_value` — both strings that will contain base64-encoded random bytes:

```rust
use super::*;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Secret {
    key: String,
    inital_value: String,
}

impl Secret {
    pub fn new(out_path: String) {
        let mut key = [0u8; 16];
        thread_rng().fill_bytes(&mut key[..]);

        let mut inital_value = [0u8; 16];
        thread_rng().fill_bytes(&mut inital_value[..]);

        let secret = Secret {
            key: base64::encode(key),
            inital_value: base64::encode(inital_value),
        };

        let secret = to_vec(&secret).unwrap();
        let mut file = File::create(out_path).unwrap();
        file.write_all(&secret).unwrap();

        println!("Secret Generated Successfully")
    }
}
```

Several invariants are upheld in this function:

1. **Key length is exactly 16 bytes.** The array `[0u8; 16]` is a fixed-size buffer. You cannot accidentally generate a 15-byte or 17-byte key — the size is a compile-time constant. This is a structural invariant enforced by the type system.
2. **Randomness comes from a cryptographically suitable source.** `thread_rng()` provides a CSPRNG (cryptographically secure pseudo-random number generator). Using a weaker source would violate the invariant that the key is unpredictable.
3. **Encoding is deterministic.** Base64 encoding ensures the binary key can be stored as a string in JSON without data loss. The invariant: `decode(encode(bytes)) == bytes` — the round-trip must be lossless.
4. **The secret is serialized as valid JSON.** Using `serde_json::to_vec` guarantees well-formed output. The file is either written completely (`write_all`) or not at all.

## Wiring Up the Main Function

Update `main.rs` to dispatch the `generate` command:

```rust
use clap::{Parser, Subcommand};
use generate::Secret;
use rand::*;
mod commands;
use commands::Cryptifer;
mod generate;
use serde_derive::{Deserialize, Serialize};
use serde_json::to_vec;
use std::fs::File;
use std::io::Write;


fn main() {
    let cryptifer = Cryptifer::parse();

    match cryptifer.command {
        commands::Commands::Generate { output_path } => {
            Secret::new(output_path);
        }
        commands::Commands::Encrypt {
            file_path: _,
            key_path: _,
        } => todo!(),
        commands::Commands::Decrypt {
            encrypted_file: _,
            key_path: _,
        } => todo!(),
    }
}
```

The `match` is exhaustive — Rust requires every variant of `Commands` to be handled. The remaining commands use `todo!()`, which will panic if called. This is an intentional invariant: **unimplemented paths fail loudly rather than silently doing nothing**. We will replace these in subsequent posts.

## Running the Generate Command

```
cargo run generate -o Keystore.secret
```

This produces a file `Keystore.secret` containing JSON with the base64-encoded key and initial value. The output looks something like:

```json
{"key":"aBcDeFgHiJkLmNoPqRsTuA==","inital_value":"xYzAbCdEfGhIjKlMnOpQrS=="}
```

The exact values will differ on every run — that is the randomness invariant at work.

## Wrapping Up

The `generate` command upholds a chain of invariants: the key is the right length, the randomness is cryptographically strong, the encoding is lossless, and the output file is valid JSON. Break any link in this chain and the entire encryption system downstream becomes unreliable.

In the next post, we will implement the `encrypt` command, which introduces the central invariant of any encryption system: **the ciphertext, when decrypted with the same key, must produce the original plaintext**.
