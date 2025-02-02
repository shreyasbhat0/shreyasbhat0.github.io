---
title: "Cryptography: Invariants Over Untrusted Channels"
date: 2025-01-05 10:00:00 +0530
categories: [Cryptography, Security]
tags: [cryptography, encryption, ciphers, invariants]
description: "Cryptography is the discipline of preserving invariants over untrusted channels. Confidentiality, integrity, authenticity — these are guarantees that must hold no matter what."
---

Every communication channel is hostile. Messages get intercepted, altered, forged. The internet routes your data through machines you don't control, operated by people you don't know, in jurisdictions you've never heard of. The fundamental problem isn't *how* to send data. It's how to send data while **preserving guarantees** about who can read it, whether it was tampered with, and who actually sent it.

These guarantees are invariants. Cryptography is the discipline of enforcing them.

## What is Cryptography?

Cryptography is a technique of securing information from unintended observers. The core mechanism is **encryption** — converting information into an incomprehensible format to maintain confidentiality.

But confidentiality is just one invariant among several. A mature cryptographic system preserves multiple properties simultaneously:

- **Confidentiality** — only authorized parties can read the data
- **Integrity** — the data has not been altered in transit
- **Authenticity** — the data actually came from who it claims to come from
- **Non-repudiation** — the sender cannot later deny having sent the data

Each of these is an invariant: a property that must hold true regardless of what an adversary does. Break any one of them, and the system has failed — even if the others still hold. Confidentiality without integrity means an attacker can't read your message, but they can flip bits in it. Integrity without authenticity means the message wasn't tampered with, but you don't know who wrote it.

## What are Ciphers?

Ciphers are the algorithms — the encryption systems — used for encrypting and decrypting data. A cipher takes plaintext and a secret key as input and produces ciphertext as output. The invariant a cipher must uphold: **given only the ciphertext (and no key), the original plaintext is computationally infeasible to recover**.

Ciphers are classified into two types based on how they process data:

- **Block Ciphers** — operate on fixed-size blocks of data (e.g., 128 bits at a time)
- **Stream Ciphers** — operate on individual bits or bytes in a continuous stream

And into three categories based on the keys used:

### Symmetric Key Ciphers

The same key encrypts and decrypts. The invariant here is **key secrecy** — if both parties keep the key secret, confidentiality holds. The moment the key leaks, every message ever encrypted with it is compromised. Past, present, future — all of it.

Examples: AES, DES, ChaCha20.

### Asymmetric Key Ciphers

The encryption key differs from the decryption key. You publish one (the public key) and guard the other (the private key). The invariant: **knowing the public key does not reveal the private key**. This is grounded in mathematical hardness assumptions — factoring large primes (RSA), discrete logarithms (Diffie-Hellman), elliptic curve problems (ECDSA).

If those hardness assumptions ever break (say, by a sufficiently powerful quantum computer), the invariant breaks with them. Every system built on that assumption fails simultaneously. This is why cryptographers treat hardness assumptions with the same seriousness that distributed systems engineers treat consensus guarantees.

### Hash Functions

No key at all. A hash function takes arbitrary input and produces a fixed-length output. The invariants:

- **Pre-image resistance** — given a hash output, you can't find an input that produces it
- **Collision resistance** — you can't find two different inputs that produce the same output
- **Avalanche effect** — a tiny change in input produces a drastically different output

When a hash function's collision resistance breaks (as happened with MD5 and SHA-1), it doesn't just weaken the algorithm — it invalidates every system that assumed collisions were impossible. Digital signatures, certificate chains, integrity checks — all of them relied on that invariant.

## The Pattern

Cryptography isn't a bag of tricks. It's a discipline of identifying which properties must always hold true, then building mathematical structures that enforce those properties against adversaries who are actively trying to violate them.

Every cipher, every protocol, every key exchange — they all start with the same question: **what must remain true, no matter what?** The answer to that question is the invariant. The algorithm is just the mechanism that preserves it.

---

*Next up: the One-Time Pad — the only cipher with a mathematical proof of perfect secrecy, and why its invariants are too expensive to maintain in practice.*
