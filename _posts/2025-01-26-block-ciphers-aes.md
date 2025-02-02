---
title: "AES Internals: Four Operations, Four Invariants"
date: 2025-01-26 10:00:00 +0530
categories: [Cryptography, Security]
tags: [cryptography, aes, block-ciphers, substitution-permutation-network, invariants]
description: "AES is built from four operations, each preserving a specific invariant. Remove any one of them and the cipher breaks in a distinct, predictable way."
---

AES won the competition to replace DES not by accident, but because its design is transparent and each component has a clear cryptographic purpose. Every operation in AES exists to preserve a specific invariant. Understanding what each operation does — and what breaks when it's removed — reveals the architecture of a well-designed cipher.

## How AES Processes Data

AES processes blocks of 128 bits (16 bytes). Rather than treating the block as a flat array, AES arranges it as a 4x4 matrix of bytes, called the **state**:

```
| s0  s4  s8  s12 |
| s1  s5  s9  s13 |
| s2  s6  s10 s14 |
| s3  s7  s11 s15 |
```

The 16-byte plaintext `(s0, s1, s2, ..., s15)` fills this matrix column by column. AES then transforms this state through multiple rounds, operating on bytes, rows, and columns to produce the ciphertext.

This 2D arrangement isn't arbitrary. It allows AES to apply transformations along two independent axes (rows and columns), achieving full diffusion — where every byte of output depends on every byte of input — in fewer rounds than a 1D structure would require.

## The Four Operations

Each AES round applies four operations to the state. Together, they uphold the cipher's core invariant: **the output is indistinguishable from a random permutation to any efficient adversary**.

### AddRoundKey

XORs a round key into the state.

This is the only operation that mixes the secret key into the data. Its invariant: **the transformation depends on the key**. Without AddRoundKey, the cipher becomes a fixed, key-independent permutation — anyone could decrypt any ciphertext without knowing the key, because the "encryption" would be a public function.

### SubBytes

Replaces each byte in the state with a corresponding byte from an S-box (substitution box).

The S-box is a fixed 256-entry lookup table, carefully chosen to be **non-linear**. The invariant SubBytes preserves: **the relationship between input and output cannot be expressed as a system of linear equations**.

Why this matters: ShiftRows and MixColumns are both linear operations. AddRoundKey is linear (XOR). If SubBytes were also linear, the entire cipher would be a single large linear transformation — a system of equations over GF(2) that could be solved with Gaussian elimination. A 128-bit key would provide essentially zero security. SubBytes is the operation that makes AES cryptographically strong rather than algebraically trivial.

### ShiftRows

Cyclically shifts the i-th row of the state matrix left by i positions:
- Row 0: no shift
- Row 1: shift left by 1
- Row 2: shift left by 2
- Row 3: shift left by 3

The invariant: **bytes from different columns are mixed together**. Without ShiftRows, the MixColumns operation would only ever combine bytes within the same column. Each column would be encrypted independently of every other column — effectively four parallel 32-bit ciphers instead of one 128-bit cipher. An attacker could break each column separately, reducing the work from 2^128 to 4 * 2^32, which is trivially feasible.

### MixColumns

Applies a linear transformation to each column of the state, mixing all four bytes within a column.

The invariant: **a change in one byte of a column affects all four bytes of that column**. Without MixColumns, the state of one byte would never influence other bytes in the same column. A chosen-plaintext attacker could construct 16 independent lookup tables (one per byte position), each with only 256 entries. Recovering the mapping would require at most 256 chosen plaintexts per byte — a trivially small number.

## Why Every Operation is Necessary

Each of the four operations addresses a specific weakness that would exist without it:

| Operation | What breaks without it |
|:----------|:----------------------|
| **AddRoundKey** | Encryption is key-independent; anyone can decrypt |
| **SubBytes** | Cipher is a linear system; solvable by algebra |
| **ShiftRows** | Columns are independent; 128-bit cipher reduces to four 32-bit ciphers |
| **MixColumns** | Bytes within a column are independent; broken by 16 small lookup tables |

This is a pattern worth noting: **each invariant guards against a specific, distinct class of attack**. The cipher's security isn't one monolithic property — it's a conjunction of multiple independent invariants, each enforced by a different operation. Break any one of them and the cipher fails, but it fails in a specific, predictable way.

## Key Expansion

AES doesn't use the same key material in every round. The master key is expanded through a **key schedule** into distinct round keys. As discussed in the previous post, using identical round keys would make the cipher vulnerable to slide attacks. The key schedule ensures each round operates with different key material, breaking the structural self-similarity that slide attacks exploit.

The key expansion invariant: **every round key is derived from the master key, but no two round keys are the same, and the derivation is non-invertible** (knowing one round key doesn't easily reveal others).

## How Decryption Works

Decryption unwinds each operation by applying its inverse in reverse order:

- **InvSubBytes** — applies the inverse S-box lookup table
- **InvShiftRows** — shifts rows in the opposite direction (right instead of left)
- **InvMixColumns** — applies the inverse of the column mixing transformation
- **AddRoundKey** — unchanged, because XOR is its own inverse (`a XOR b XOR b = a`)

The fact that AddRoundKey is self-inverse is a consequence of XOR's algebraic property and it simplifies implementation. The other three operations each require separate inverse implementations, which means AES decryption is a distinct code path from encryption — unlike Feistel ciphers (like DES) where encryption and decryption share the same structure.

## AES as an Invariant System

What makes AES interesting from an invariant perspective is that it's a layered composition of guarantees. No single operation makes the cipher secure. Security emerges from the combination:

1. SubBytes provides non-linearity
2. ShiftRows provides inter-column diffusion
3. MixColumns provides intra-column diffusion
4. AddRoundKey provides key-dependence
5. Multiple rounds compound these effects until full diffusion is achieved

After just two rounds, every output bit depends on every input bit. After the full 10-14 rounds, the relationship between plaintext and ciphertext is so complex that no shortcut faster than brute force has been found — despite 25+ years of intense public analysis.

The design principle: **build security not from one strong invariant, but from multiple independent invariants that reinforce each other**. This is the same principle behind defense in depth, layered network security, and the Swiss cheese model of failure prevention. Any one layer might have holes. The holes are unlikely to align.

---

*Next: Block cipher modes of operation — because encrypting a single 128-bit block is the easy part. The hard part is encrypting real messages that are longer than 128 bits without leaking information.*
