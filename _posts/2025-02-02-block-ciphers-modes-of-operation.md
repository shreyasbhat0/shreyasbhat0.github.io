---
title: "Modes of Operation: Same Cipher, Different Invariants"
date: 2025-02-02 10:00:00 +0530
categories: [Cryptography, Security]
tags: [cryptography, block-ciphers, ecb, cbc, ctr, modes-of-operation, invariants]
description: "A block cipher encrypts fixed-size blocks. Modes of operation define how to encrypt messages longer than one block — and each mode upholds (or fails to uphold) different invariants."
---

A block cipher like AES encrypts exactly 128 bits at a time. Real messages are longer than 128 bits. The question of *how* to apply a block cipher to multi-block messages is the question of **modes of operation** — and the choice of mode determines which invariants hold and which ones break.

Three modes are fundamental:

- **Electronic Codebook (ECB)** — the simple, broken one
- **Cipher Block Chaining (CBC)** — the chained one
- **Counter Mode (CTR)** — the parallelizable one

Each makes a different tradeoff. Understanding where each fails is more instructive than understanding how each works.

## Electronic Codebook Mode (ECB)

ECB is the most obvious approach: encrypt each block independently.

```
C1 = E(K, P1)
C2 = E(K, P2)
...
Cn = E(K, Pn)
```

Decryption reverses each block independently:

```
P1 = D(K, C1)
P2 = D(K, C2)
...
Pn = D(K, Cn)
```

### The Invariant ECB Violates

ECB preserves confidentiality at the individual block level — given a single ciphertext block, you can't recover the plaintext without the key. But it violates a much more important invariant: **identical plaintexts must produce distinct ciphertexts**.

In ECB, if `P3 = P7`, then `C3 = C7`. Always. The ciphertext reveals patterns in the plaintext. An attacker who sees repeated ciphertext blocks knows the corresponding plaintext blocks were identical — even without knowing what those blocks contain.

The classic demonstration: encrypt a bitmap image with ECB. The pixel data changes, but the image structure remains visible because repeated color blocks produce repeated ciphertext blocks. The outline of the image is clearly recognizable in the encrypted output. You've encrypted every byte, but you've hidden nothing about the structure.

This is a violation of **semantic security** — the invariant that an attacker cannot learn *any* property of the plaintext from the ciphertext. ECB leaks block-level equality, which is often enough to reveal the nature of the data.

**ECB should not be used for encrypting data longer than a single block.** It exists as a building block and a cautionary example, not as a practical mode.

## Cipher Block Chaining (CBC)

CBC fixes ECB's pattern leakage by **chaining** blocks together. Instead of encrypting each block in isolation, each plaintext block is XORed with the previous ciphertext block before encryption:

```
C1 = E(K, P1 XOR IV)
C2 = E(K, P2 XOR C1)
C3 = E(K, P3 XOR C2)
...
Cn = E(K, Pn XOR Cn-1)
```

The first block has no predecessor, so CBC introduces an **Initialization Vector (IV)** — a random value that serves as a synthetic "previous ciphertext block" for the first plaintext block.

### The Invariants CBC Upholds

**Identical plaintext blocks produce different ciphertext blocks.** Because each block depends on all previous blocks, even identical plaintext blocks at different positions encrypt to different ciphertext. The pattern leakage of ECB is eliminated.

**The IV guarantees that two identical messages encrypt differently.** If Bob sends "ATTACK AT DAWN" twice with different random IVs, the two ciphertexts will be completely different. An attacker observing the channel can't tell that the same message was sent twice.

### The Invariant CBC Introduces

CBC creates a new constraint: **encryption is sequential**. Block `Cn` depends on `Cn-1`, which depends on `Cn-2`, and so on. You must encrypt blocks in order. There is no way to parallelize CBC encryption.

Decryption, however, *is* parallelizable. To decrypt block `Cn`, you need `Cn-1` (the previous ciphertext block) and the key. All ciphertext blocks are available from the start, so every block can be decrypted simultaneously.

```
P1 = D(K, C1) XOR IV
P2 = D(K, C2) XOR C1
...
Pn = D(K, Cn) XOR Cn-1
```

This asymmetry — sequential encryption, parallel decryption — matters in practice. Systems that need to encrypt large amounts of data quickly (streaming, real-time protocols) pay a performance cost with CBC.

### CBC's Vulnerability: The IV

The IV must be **unpredictable**. If an attacker can predict the IV for the next message, they can construct chosen-plaintext attacks that break the semantic security guarantee. This was the basis of the BEAST attack against TLS 1.0, which used predictable IVs in CBC mode. The invariant isn't just "the IV must be random" — it's "the IV must be unpredictable *to the attacker at the time of encryption*."

## Counter Mode (CTR)

CTR mode takes a fundamentally different approach. Rather than chaining blocks together, it transforms the block cipher into a **stream cipher** — it doesn't encrypt the plaintext directly at all.

Instead, CTR mode encrypts a sequence of counter blocks to generate a keystream, then XORs that keystream with the plaintext:

```
Keystream1 = E(K, Nonce || Counter1)
Keystream2 = E(K, Nonce || Counter2)
...
KeystreamN = E(K, Nonce || CounterN)

C1 = P1 XOR Keystream1
C2 = P2 XOR Keystream2
...
Cn = Pn XOR KeystreamN
```

The **nonce** (number used once) is a value that is the same for all blocks within a single message. The **counter** is an integer that increments for each block.

### The Invariants CTR Upholds

**Full parallelism.** Every keystream block can be computed independently — you just need the key, the nonce, and the counter value. Encryption and decryption are both fully parallelizable. You can even start generating the keystream *before* you have the plaintext.

**No padding required.** Because CTR mode produces a stream of bytes that is XORed with the plaintext, the message doesn't need to be a multiple of the block size. You just truncate the keystream to match the message length. This eliminates an entire class of padding oracle attacks that have plagued CBC implementations.

**Decryption is identical to encryption.** XOR is its own inverse. The same code path handles both directions.

### The Critical Invariant: Nonce Uniqueness

The security of CTR mode rests on one absolute invariant: **the (nonce, counter) pair must never repeat for the same key**.

If the same nonce is used for two different messages under the same key, the same keystream is generated for both. And we've seen this failure before — it's exactly the One-Time Pad key reuse problem:

```
C1 XOR C2 = (P1 XOR Keystream) XOR (P2 XOR Keystream) = P1 XOR P2
```

The keystream cancels out, and the attacker gets the XOR of two plaintexts. With CTR mode, a nonce reuse completely destroys confidentiality. Not weakens it. Destroys it.

The nonce doesn't need to be random — it just needs to be unique. A simple counter works fine, as long as it never wraps around and is never reset. But "never reuse this value" is an invariant that requires discipline. Databases, restarts, key rotations, distributed systems with multiple encrypters — all of these create opportunities for accidental nonce reuse.

### CTR vs. CBC: The Tradeoff

| Property | CBC | CTR |
|:---------|:----|:----|
| Parallel encryption | No | Yes |
| Parallel decryption | Yes | Yes |
| Requires padding | Yes | No |
| Random access to blocks | No | Yes |
| Critical fragility | Predictable IV | Nonce reuse |

Both modes uphold semantic security. Both eliminate ECB's pattern leakage. The difference is in their failure modes and their operational requirements.

## The Broader Pattern

Modes of operation illustrate a recurring theme in system design: **the same primitive can be composed in different ways, and the composition determines which invariants hold**.

AES is the same cipher in ECB, CBC, and CTR. The block-level encryption is identical. But the mode — the way blocks relate to each other — determines whether patterns are leaked, whether encryption is parallelizable, and what operational invariant (unique IV, unique nonce) must be maintained.

This is true far beyond cryptography. The same database engine behaves differently under different isolation levels. The same consensus algorithm provides different guarantees depending on the network model. The same programming language offers different safety properties depending on which features you use (Rust with `unsafe` is a different language from Rust without it).

The primitive doesn't define the system's invariants. The composition does.

---

*This post concludes the cryptography fundamentals series. The building blocks — symmetric encryption, block ciphers, and modes of operation — are the foundation for everything that follows: authenticated encryption, key exchange, digital signatures, and the protocols (TLS, Signal, blockchain consensus) that compose them into real systems.*
