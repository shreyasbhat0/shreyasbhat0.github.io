---
title: "Building a CLI App in Rust, Part 3: Encryption and the Invariant of Confidentiality"
date: 2025-06-28 10:00:00 +0530
categories: [Rust, CLI]
tags: [rust, clap, cli, invariants, cryptography, aes, encryption]
description: "Encryption is an invariant transformation: given a key K and plaintext P, the ciphertext C must be deterministic, reversible with K, and indistinguishable from random data without K. The encrypt command must uphold all three properties."
---

In the previous post, we implemented the `generate` command, which produces a cryptographic key and IV with strict invariants on length, randomness, and encoding. Now we put that secret to use.

The `encrypt` command takes a file and a secret, and transforms the file contents into ciphertext. The core invariant of encryption is deceptively simple: **`decrypt(encrypt(plaintext, key), key) == plaintext`**. This is the round-trip invariant — the same one that governs every serialization format, every codec, every reversible transformation. Break it, and the data is gone.

But encryption adds a second invariant that serialization does not require: **without the key, the ciphertext reveals nothing about the plaintext**. This is the confidentiality invariant. It constrains not just correctness but security.

We will use AES (Advanced Encryption Standard) in CBC (Cipher Block Chaining) mode with PKCS7 padding.

## Adding Dependencies

Add the encryption libraries to `Cargo.toml`:

```
aes = "0.7.4"
block-modes = "0.8.1"
```

Import them in `main.rs`:

```rust
use aes::Aes256;
use block_modes::{block_padding::Pkcs7, Cbc};
```

Define a type alias that encodes our cipher configuration:

```rust
pub type AES = Cbc<Aes128, Pkcs7>;
```

This single line establishes several invariants at the type level:
- The cipher is AES-128 (128-bit key).
- The mode of operation is CBC (each block depends on the previous one — a single-bit change in the plaintext changes all subsequent ciphertext blocks).
- The padding scheme is PKCS7 (the plaintext is padded to align with the block size, and the padding is always removable during decryption).

These are not runtime choices. They are compile-time constants. You cannot accidentally use ECB mode or forget padding — the type system prevents it.

## Implementing the Encrypt Function

The encryption function reads the target file and the secret, then transforms the file contents:

```rust
pub fn encrypt(file_path: String, key_path: String) {
    let to_encrypt = read(file_path.clone()).unwrap();
    let secret = read(key_path).unwrap();
}
```

We use the standard filesystem library to read both files. Next, we deserialize the secret to extract the key and IV:

```rust
pub fn encrypt(file_path: String, key_path: String) {
    let to_encrypt = read(file_path.clone()).unwrap();
    let secret = read(key_path).unwrap();
    let secret: Secret = serde_json::from_slice(&secret).unwrap();
}
```

The deserialization step enforces an invariant: **the secret file must contain valid JSON that matches the `Secret` struct**. If someone manually edits the file and corrupts the JSON, `from_slice` fails rather than producing a malformed key.

Create the cipher instance:

```rust
pub fn encrypt(file_path: String, key_path: String) {
    let to_encrypt = read(file_path.clone()).unwrap();
    let secret = read(key_path).unwrap();
    let cipher = AES::new_from_slices(&secret.decode_key(), &secret.decode_iv()).unwrap();
}
```

`new_from_slices` validates that the key and IV are the correct length for AES-128. If the base64 decoding produced the wrong number of bytes, this call fails. Another invariant enforced at the boundary.

Now allocate a buffer, encrypt, and write back:

```rust
pub fn encrypt(file_path: String, key_path: String) {
    let to_encrypt = read(file_path.clone()).unwrap();
    let secret = read(key_path).unwrap();

    let secret: Secret = serde_json::from_slice(&secret).unwrap();
    let cipher = AES::new_from_slices(&secret.decode_key(), &secret.decode_iv()).unwrap();

    let pos = to_encrypt.len();

    let mut buffer = vec![0u8; pos + pos];
    buffer[..pos].copy_from_slice(&to_encrypt);

    let encrypted_data = cipher.encrypt(&mut buffer, pos).unwrap();

    write(file_path, base64::encode(encrypted_data)).unwrap();
}
```

The buffer is twice the size of the input to accommodate PKCS7 padding. In the worst case, padding adds up to one full block (16 bytes), but allocating `2 * pos` provides a safe margin. The invariant: **the buffer is always large enough to hold the padded ciphertext**. An undersized buffer would cause `encrypt` to fail or, worse, truncate data silently.

The final ciphertext is base64-encoded before writing. This ensures the encrypted file contains only printable ASCII characters — important for tools that might process the file downstream.

## Updating the Main Function

Wire the `encrypt` command into the dispatcher:

```rust
fn main() {
    let cryptifer = Cryptifer::parse();

    match cryptifer.command {
        commands::Commands::Generate { output_path } => {
            Secret::new(output_path);
        }
        commands::Commands::Encrypt {
            file_path,
            key_path,
        } => {
            encrypt(file_path, key_path);
        }
        commands::Commands::Decrypt {
            encrypted_file: _,
            key_path: _,
        } => todo!(),
    }
}
```

## Running the Encrypt Command

```
cargo run encrypt  -f file.txt -k Keystore.secret
```

After running this, the contents of `file.txt` are replaced with the base64-encoded ciphertext. The original plaintext is gone from the file — it exists only in encrypted form now.

This is worth pausing on. The encrypt command is *destructive* — it overwrites the input file. This is a design choice with its own invariant implications: after encryption, the only way to recover the plaintext is through the decrypt command with the correct key. If the key is lost, the data is lost. That is not a bug. That is the confidentiality invariant working as intended.

## The Chain of Invariants So Far

Looking at the system as a whole, we now have a chain:

1. **Clap** ensures the arguments are present and well-formed.
2. **`generate`** ensures the key is the right length, random, and properly encoded.
3. **`encrypt`** ensures the ciphertext is produced by a correctly-configured cipher with a valid key.

Each invariant depends on the one before it. If `generate` produced a 15-byte key, `encrypt` would fail at `new_from_slices`. If Clap allowed a missing `--key-path`, `encrypt` would fail at file read. The invariants form a chain of trust — each layer validates its inputs and produces outputs that satisfy the next layer's preconditions.

## Wrapping Up

We now have two working commands. The `encrypt` command upholds the confidentiality invariant: without the key, the ciphertext is meaningless. In the next post, we will implement `decrypt` and close the loop — proving that the round-trip invariant `decrypt(encrypt(P, K), K) == P` actually holds.
