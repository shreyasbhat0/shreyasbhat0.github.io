---
title: "Block Ciphers"
date: 2025-01-19 10:00:00 +0530
categories: [Cryptography, Security]
tags: [cryptography, block-ciphers, des, aes, feistel-network, invariants]
description: "Block ciphers are the workhorses of modern encryption. They trade the mathematical perfection of the One-Time Pad for practical invariants that can actually be maintained at scale."
---

The One-Time Pad gives you perfect secrecy but demands perfect discipline — keys as long as messages, true randomness, no reuse, absolute secrecy. In practice, those invariants are too expensive to maintain. Block ciphers take a different approach: use a short, fixed-length key to encrypt data of arbitrary length, and rely on *computational hardness* rather than information-theoretic perfection.

The invariant changes from "an attacker learns nothing" to "an attacker cannot learn anything in any reasonable amount of time." That's a weaker guarantee in theory, but a usable one in practice.

## What are Block Ciphers?

A block cipher takes a fixed-size block of input (n bits) and produces a fixed-size block of output (n bits), parameterized by a secret key.

The encryption function E takes a key K and a plaintext block P, producing ciphertext C:

```
C = E(K, P)
```

The decryption function D takes the same key K and the ciphertext C, recovering the original plaintext:

```
P = D(K, C)
```

The fundamental invariant: **for any key K, the encryption function E(K, .) is a bijection** — a one-to-one mapping from plaintext blocks to ciphertext blocks. Every plaintext maps to exactly one ciphertext, and every ciphertext maps back to exactly one plaintext. If this invariant broke, decryption would be ambiguous — two different plaintexts could encrypt to the same ciphertext, and the receiver wouldn't know which was intended.

## How Block Ciphers Work: Iterated Rounds

Block ciphers are built by iteration. The key K is expanded into a sequence of round keys (k1, k2, ..., kn), and the plaintext is transformed through multiple applications of a round function R.

For a block cipher with three rounds:

```
C = R(k3, R(k2, R(k1, P)))
```

Each round function takes a round key derived from the master key and applies it to the current state. The security of the cipher comes from the *composition* of rounds — each round individually might be weak, but stacked together they produce a transformation that is computationally indistinguishable from a random permutation.

This is an invariant about the cipher's design: **after a sufficient number of rounds, the output must be indistinguishable from random to any efficient algorithm that doesn't know the key**. This property is called pseudorandomness, and it's what separates a secure block cipher from a fancy substitution table.

### Round Keys Must Not Be Identical

An important design invariant: **round keys should not be identical across rounds**. If every round uses the same key, the cipher becomes vulnerable to a slide attack.

#### Slide Attack

A slide attack exploits structural self-similarity. If all rounds are identical (same round function, same key), then the attacker looks for two plaintext-ciphertext pairs (P1, C1) and (P2, C2) where:

```
P2 = R(k, P1)
```

If the round function R is the same at every round, this relationship propagates through the entire cipher:

```
C2 = R(k, C1)
```

This holds regardless of the number of rounds. The cipher could have 10 rounds or 10,000 — the slide relationship still holds because the structure is self-similar. By finding such a "slid pair," the attacker can recover the round key with far less work than brute force.

The fix is straightforward: derive distinct round keys from the master key using a key schedule. Each round looks different, breaking the self-similarity that the slide attack depends on. The invariant is: **no two rounds should be structurally identical**.

## Two Notable Block Ciphers

### DES: Data Encryption Standard

DES was designed by an engineer at IBM and adopted as a federal standard in 1977. It operates on 64-bit blocks with a 56-bit key.

DES uses a **Feistel network** structure, which has an elegant property: the encryption and decryption algorithms are nearly identical — you just reverse the order of the round keys.

The Feistel round works as follows:

1. Split the 64-bit block into two 32-bit halves, L and R
2. Compute `L = L XOR F(R)`, where F is a substitution-permutation round function
3. Swap L and R
4. Repeat steps 2-3 for 16 rounds total
5. Merge L and R into the 64-bit output block

The Feistel structure preserves the bijection invariant by construction. Because only half the block is modified in each round (and the modification is XORed, which is always reversible), the transformation is guaranteed to be invertible. You don't need to prove that the round function F is invertible — the structure ensures invertibility regardless of what F does.

However, DES has a critical weakness: its 56-bit key. In 1998, the Electronic Frontier Foundation built a machine for $250,000 that could brute-force a DES key in about 56 hours. The invariant "brute force is infeasible" no longer held. DES was officially deprecated.

### AES: Advanced Encryption Standard

The Rijndael cipher, developed by Belgian cryptographers Joan Daemen and Vincent Rijmen, won a five-year public competition to replace DES. It became the AES (Advanced Encryption Standard) and remains the dominant symmetric cipher today.

AES processes 128-bit blocks using keys of 128, 192, or 256 bits. The number of rounds depends on the key size:

| Key Size | Rounds |
|:---------|:-------|
| 128 bits | 10     |
| 192 bits | 12     |
| 256 bits | 14     |

Unlike DES, AES uses a **Substitution-Permutation Network (SPN)** rather than a Feistel structure. Every bit of the block is transformed in every round (not just half), which means AES achieves full diffusion faster — fewer rounds needed for the same security margin.

The invariant AES aims to uphold: **the cipher behaves as a pseudorandom permutation under the chosen key**. After 25+ years of public cryptanalysis by the world's best researchers, no practical attack on full AES has been found. The best known attacks are only marginally better than brute force and require impractical amounts of data and computation.

## The Engineering Tradeoff

Block ciphers represent a deliberate tradeoff. The One-Time Pad's invariant (perfect secrecy) requires invariants on the key that are impractical to maintain. Block ciphers weaken the secrecy guarantee from "information-theoretically perfect" to "computationally infeasible to break," but in exchange, the preconditions become manageable:

- The key is short (128-256 bits) and can be transmitted securely
- The key can encrypt many messages (with appropriate modes of operation)
- No true randomness is needed for the encryption itself — only for key generation

The invariant shifts from "this cannot be broken" to "this cannot be broken *by any known method in any reasonable time*." That's a guarantee conditioned on our understanding of computational complexity. If P = NP, or if quantum computers scale far enough, some of these guarantees will need revisiting. But for now, they hold — and unlike the OTP, they hold in practice, not just in theory.

---

*Next: a deep dive into how AES actually transforms data — the four operations that make it work, and what happens when any one of them is removed.*
