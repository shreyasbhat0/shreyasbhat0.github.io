---
title: "Building a CLI App in Rust, Part 4: Decryption and the Round-Trip Invariant"
date: 2025-07-12 10:00:00 +0530
categories: [Rust, CLI]
tags: [rust, clap, cli, invariants, cryptography, aes, decryption]
description: "The decrypt command closes the loop. The round-trip invariant — decrypt(encrypt(plaintext, key), key) == plaintext — is the single property that proves the entire system works. If it fails, nothing else matters."
---

We have been building up to this moment. In Part 1, we established input validation invariants with Clap. In Part 2, we enforced key generation invariants — correct length, sufficient randomness, lossless encoding. In Part 3, we encrypted files with AES-CBC, maintaining the confidentiality invariant.

Now we close the loop with the `decrypt` command. This is where the most important invariant of the entire system is tested: **`decrypt(encrypt(plaintext, key), key) == plaintext`**.

This is the round-trip invariant. It is not just a nice property — it is the *definition* of a correct encryption system. If this invariant fails, the encrypt command is not encrypting; it is destroying data. Every design decision we have made so far — the key length, the padding scheme, the base64 encoding, the cipher mode — must be exactly consistent between encrypt and decrypt for the round-trip to hold.

## Implementing the Decrypt Command

Create a new file `decrypt.rs` in the `src` directory. Add it to the module tree in `main.rs`:

```rust
mod decrypt;
use decrypt::decrypt;
```

Now implement the decryption function:

```rust
use super::*;

pub fn decrypt(encrypted_file_path: String, secret: String) {
    let encrypted_content = read(encrypted_file_path.clone()).unwrap();
    let secret = read(secret).unwrap();
    let secret: Secret = serde_json::from_slice(&secret).unwrap();

    let cipher = AES::new_from_slices(&secret.decode_key(), &secret.decode_iv()).unwrap();

    let mut buffer = base64::decode(encrypted_content).unwrap();
    let decrypted_data = cipher.decrypt(&mut buffer).unwrap();

    write(encrypted_file_path, str::from_utf8(decrypted_data).unwrap()).unwrap();
}
```

Every line in this function mirrors a step in `encrypt`, but in reverse. This symmetry is not accidental — it is a direct consequence of the round-trip invariant. Let us trace the correspondence:

| Encrypt | Decrypt |
|:--|:--|
| Read plaintext from file | Read ciphertext from file |
| Read secret, deserialize to `Secret` | Read secret, deserialize to `Secret` |
| Create AES cipher from key and IV | Create AES cipher from *same* key and IV |
| Encrypt plaintext, base64-encode result | Base64-decode ciphertext, decrypt result |
| Write ciphertext to file | Write plaintext to file |

The invariant is maintained by ensuring that every transformation in `encrypt` has an exact inverse in `decrypt`:

1. **Base64 encoding / decoding.** `encrypt` encodes the ciphertext before writing; `decrypt` decodes it before decrypting. The invariant: `decode(encode(data)) == data`. If the encrypted file is modified — even a single character changed — `base64::decode` will either fail or produce corrupted bytes, and the cipher will reject them.

2. **AES-CBC encrypt / decrypt.** The cipher instance is constructed from the same key and IV in both functions. CBC mode requires this exact match — using a different IV produces different output blocks, violating the round-trip. PKCS7 padding is applied during encryption and stripped during decryption. If the padding is malformed (because the key was wrong, or the ciphertext was tampered with), `cipher.decrypt` returns an error rather than garbage. This is an integrity check built into the padding scheme.

3. **`Secret` deserialization.** Both functions deserialize the secret file identically. The invariant: the same `Secret` struct, produced by `generate`, is used by both encrypt and decrypt. If someone generates a new secret between encrypting and decrypting, the round-trip breaks — by design. The key is the invariant link between the two operations.

## Updating the Main Function

Complete the match block:

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
            encrypted_file,
            key_path,
        } => decrypt(encrypted_file, key_path),
    }
}
```

There are no more `todo!()` calls. Every variant of `Commands` now has an implementation. The exhaustive match invariant — "all cases are handled" — is fully satisfied.

## Running the Decrypt Command

```
cargo run decrypt  -f <encrypted_file_path> -k <secret_file_path>
```

This reads the encrypted file, decrypts it using the secret, and writes the original plaintext back to the file. If you encrypted `file.txt` in Part 3, running decrypt with the same secret will restore the original contents.

The full workflow:

```bash
# Generate a secret
cargo run generate -o Keystore.secret

# Encrypt a file
cargo run encrypt -f file.txt -k Keystore.secret

# file.txt now contains ciphertext

# Decrypt the file
cargo run decrypt -f file.txt -k Keystore.secret

# file.txt is back to its original contents
```

This is the round-trip invariant in action. The file goes through `plaintext -> ciphertext -> plaintext`, and the final state is identical to the initial state.

## What Breaks the Round-Trip

Understanding an invariant means understanding how it can be violated:

- **Wrong key**: Using a different secret file for decryption than for encryption. The cipher produces garbage, and PKCS7 padding validation fails.
- **Corrupted ciphertext**: If even one byte of the encrypted file is changed (by a text editor normalizing line endings, for example), base64 decoding may fail or the decrypted padding will be invalid.
- **Mismatched cipher configuration**: If `encrypt` used AES-128 but `decrypt` used AES-256, the block sizes would not match. Our type alias `AES` prevents this by fixing the configuration at the type level.
- **Lost IV**: The IV is as important as the key for CBC mode. Losing or changing it breaks decryption of the first block, which cascades to all subsequent blocks.

Each of these failure modes is a violation of a specific sub-invariant that the round-trip depends on. The system is as strong as its weakest link.

## Wrapping Up

Cryptifer is now functionally complete. We can generate keys, encrypt files, and decrypt them. The round-trip invariant — the most important property of the entire system — holds.

But there is a problem. Look at all those `.unwrap()` calls scattered through the code. Every one of them is a potential panic — an uncontrolled crash with no useful error message. The invariant "the program either succeeds or provides a meaningful error" is violated everywhere. In the next and final post, we will fix this with proper error handling, turning panics into informative messages and making the program robust against invalid input at every stage.
