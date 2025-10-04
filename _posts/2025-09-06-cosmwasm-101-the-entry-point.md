---
title: "CosmWasm 101: The Entry Points"
date: 2025-09-06 10:00:00 +0530
categories: [Blockchain, CosmWasm]
tags: [cosmwasm, rust, smart-contracts, invariants]
description: "Smart contracts don't have main(). They have entry points — and each one enforces a different invariant about how the outside world can interact with on-chain state."
---

Every smart contract makes a promise: *the only way to change my state is through me*. Not through direct storage writes. Not through backdoor admin calls. Through a well-defined set of **entry points** that enforce the contract's invariants at every boundary.

CosmWasm is a smart contract platform for the Cosmos SDK. Unlike EVM contracts with their single fallback dispatcher, CosmWasm makes the entry point structure explicit. Each entry point has a distinct signature, distinct capabilities, and distinct invariants it must uphold. Understanding these boundaries is the first step to writing contracts that are correct by construction.

This post covers the foundational setup: prerequisites, project structure, and the four core entry points.

---

> I assume familiarity with Rust. If you are new to the language, the [Rust Book](https://doc.rust-lang.org/book/) is the canonical starting point.

## Prerequisites

Install Rust:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Install the **wasmd** binary (the Wasm-enabled Cosmos chain daemon):

```bash
git clone git@github.com:CosmWasm/wasmd.git
cd ./wasmd
make install
```

Add the Wasm compilation target:

```bash
rustup target add wasm32-unknown-unknown
```

---

## Getting Started

Smart contracts in CosmWasm are Rust libraries, not binaries. There is no `fn main()`. The invariant here is structural: **the contract never controls its own execution lifecycle**. The chain calls into the contract, never the other way around.

Create a new library project:

```bash
cargo new cw-simple-counter --lib
```

Update `Cargo.toml` to add the CosmWasm standard library:

```ini
[package]
name = "cw-simple-counter"
version = "0.1.0"
edition = "2021"

[dependencies]
cosmwasm-std = {version = "1.2.2",features=["iterator"]}
```

The `cosmwasm-std` crate provides the structs, enums, functions, and macros used across all CosmWasm contracts. It defines the types that enforce the contract boundary invariants at the type level.

## Defining Entry Points

Rust binaries have a single entry point: `fn main()`. Smart contracts break this model deliberately. Instead of one door into the program, there are several — each with different permissions and different guarantees about what can happen to state.

CosmWasm defines four core entry points:

- **`instantiate`** — called exactly once, when the contract is first deployed. The invariant: after instantiate succeeds, the contract's initial state is valid and all subsequent operations can assume it.
- **`execute`** — handles all state-mutating operations. The invariant: every state transition produced by execute must move the contract from one valid state to another.
- **`query`** — reads state without modifying it. The invariant: **query must never mutate state**. This is enforced at the type level — query receives `Deps` (immutable), not `DepsMut`.
- **`reply`** — handles responses from sub-message execution. The invariant: reply only fires in response to a sub-message the contract itself dispatched.

Notice the asymmetry between `execute` and `query`. This is not a convention — it is a type-level guarantee. The compiler enforces the read-only invariant of queries by giving them a different dependency type. You cannot accidentally write state in a query handler because the types will not allow it.

Here are the entry points in `lib.rs`:

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Reply, Response, StdResult,
};

#[entry_point]
pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    unimplemented!()
}

#[entry_point]
pub fn execute(_deps: DepsMut, _env: Env, _info: MessageInfo, _msg: Empty) -> StdResult<Response> {
    unimplemented!()
}

#[entry_point]
pub fn query(_deps: Deps, _env: Env, _msg: Empty) -> StdResult<Binary> {
    unimplemented!()
}

#[entry_point]
pub fn reply(_deps: DepsMut, _env: Env, _msg: Reply) -> StdResult<Response> {
    unimplemented!()
}
```

## The Arguments as Invariant Carriers

Each argument to an entry point encodes a specific invariant about what information is available and what operations are permitted:

- **`DepsMut` and `Deps`** — the gateway to the outside world. `DepsMut` allows querying *and updating* contract state, querying other contracts, and accessing address-manipulation helpers. `Deps` allows only reads. The distinction between these two types is the mechanism by which CosmWasm enforces the query-never-mutates invariant at compile time.

- **`Env`** — represents the blockchain's state at the moment of execution: chain height, chain ID, current timestamp, and the contract's own address. The invariant: `Env` is populated by the runtime, not the caller. A contract can trust these values because they cannot be spoofed by a message sender.

- **`MessageInfo`** — metadata about who sent the message and what native tokens they attached. This is only present in `instantiate` and `execute`, not in `query`. The invariant: the `sender` field is authenticated by the chain. You do not need to verify signatures yourself — if `info.sender` says an address, that address signed the transaction.

- **`Empty`** — the type representing an empty JSON object `{}`. In practice, this placeholder gets replaced with contract-specific message types (`InstantiateMsg`, `ExecuteMsg`, `QueryMsg`). The type parameter is what makes each entry point handle only the messages it is designed for — another compile-time invariant.

- **`Reply`** — carries the ID and execution result of a sub-message. Only available in the `reply` entry point, binding the response to the specific sub-message that triggered it.

## The Pattern

What emerges from CosmWasm's entry point design is a principle that applies far beyond smart contracts: **every boundary in a system should make its invariants explicit in its type signature**.

The query handler cannot mutate state — not because of documentation, not because of code review, but because `Deps` does not have a `save` method. The execute handler knows who called it — not because of a convention, but because `MessageInfo` is always present and always authenticated.

When invariants live in the type system, they do not need tests. They do not need discipline. They are simply true.

---

The source code for this series is available on [GitHub](https://github.com/shreyasbhat0/cosmwasm101). In the next post, we will define the actual messages for instantiation, execution, and query — and implement the state transitions that these entry points guard.
