---
title: "CosmWasm 101: Messages and State"
date: 2025-10-04 10:00:00 +0530
categories: [Blockchain, CosmWasm]
tags: [cosmwasm, rust, smart-contracts, invariants, state-management]
description: "Instantiate, Execute, Query — three message types, three classes of invariants. How CosmWasm contracts enforce valid state transitions through typed messages and ownership checks."
---

In the [previous post](/posts/cosmwasm-101-the-entry-points/), we set up a CosmWasm contract with four entry points — each guarding a different invariant about how the outside world interacts with on-chain state. But those entry points were stubs. They accepted `Empty` messages and did nothing.

This post fills them in. We define typed messages for instantiation, execution, and query. We implement state storage. We wire up the logic that transforms messages into state transitions. Along the way, every design decision maps to an invariant — a property that must hold true before, during, and after the operation.

## The Instantiate Message: Establishing the Initial Invariant

Every contract begins with instantiation. This is the one moment where state is created from nothing — there is no previous state to validate against. The invariant that `instantiate` must establish: **after this function returns Ok, the contract's state is fully initialized and valid for all subsequent operations**.

Create `msg.rs` and define the instantiation message:

```rust
use super::*;

// Message for Instantiating Contract
#[cw_serde]
pub struct InstantiateMsg {
    pub counter: u64,
}
```

The message is simple: an initial counter value. But the type carries meaning. By making `counter` a `u64`, we establish an invariant at the type level: **the counter is always a non-negative integer**. There is no runtime check needed — the serialization layer rejects anything that is not a valid `u64` before the contract code ever runs.

## State: The Invariant That Persists

In CosmWasm, blockchain state is a massive key-value store. Each contract's keys are prefixed with metadata that scopes them to that contract alone. The invariant: **a contract can only read and write its own state**. The runtime enforces this — no contract can reach into another's storage, regardless of what code it runs.

Add the `cw-storage-plus` crate to `Cargo.toml`:

```ini
[dependencies]
cosmwasm-std = {version = "1.2.2",features=["iterator"]}
cw-storage-plus = "1.0.1"
```

Create `state.rs` and define the contract state:

```rust
use super::*;

#[cw_serde]
pub struct CwSimpleCounter {
    pub owner: String,
    pub counter: u64,
}

pub const STATE: Item<CwSimpleCounter> = Item::new("COUNTER");
```

`Item<T>` is a typed wrapper around a single key in the store. The invariant it enforces: **this key always contains a value of type `CwSimpleCounter`, or it does not exist**. There is no intermediate state. No partially written struct. No type mismatch. The serialization boundary guarantees that what you save is what you load.

Note the `owner` field. This is not just metadata — it is the foundation of an authorization invariant we will enforce in every execute handler.

## Implementing Instantiate

```rust
#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)
        .map_err(ContractError::Std)?;
    let contract_state = CwSimpleCounter {
        owner: info.sender.to_string(),
        counter: msg.counter,
    };
    STATE
        .save(deps.storage, &contract_state)
        .map_err(ContractError::Std)?;

    Ok(Response::new()
        .add_attribute("method", "instantiate")
        .add_attribute("value", msg.counter.to_string()))
}
```

Several invariants are established here:

1. **Contract version is recorded** — `set_contract_version` stores the contract name and version, which is critical during migrations. The invariant: a deployed contract always knows what version it is.
2. **The sender becomes the owner** — `info.sender` is authenticated by the chain. The invariant: the owner is always a valid, authenticated address. No one can instantiate a contract and assign ownership to someone else's address maliciously, because the chain verifies the sender.
3. **State is fully initialized** — both `owner` and `counter` are set. There is no code path where `instantiate` returns `Ok` but state is only partially written.

## Error Handling: Making Violations Explicit

Create `error.rs` for structured error types:

```rust
use super::*;

#[derive(Error, Debug)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),
    #[error("DecodeError {error}")]
    DecodeError { error: String },
}
```

Every fallible operation in the contract returns `Result<T, ContractError>`. The invariant: **no error is silently swallowed**. If state cannot be saved, if a message cannot be decoded, the entire transaction is rolled back. CosmWasm transactions are atomic — either every state change commits, or none of them do. This is the same ACID invariant that databases enforce, applied at the transaction level.

## Execution Messages: Guarded State Transitions

Execution messages are the only way to mutate contract state after instantiation. Each variant represents a specific, named state transition:

```rust
#[cw_serde]
pub enum ExecuteMsg {
    IncrementCounter {},
    DecrementCounter {},
    UpdateCounter { count: u64 },
    ResetCounter {},
}
```

The `enum` structure is itself an invariant: **the only state mutations that can happen are the ones explicitly listed here**. There is no catch-all handler. There is no way to send an arbitrary message that mutates state. The type system makes unexpected mutations unrepresentable.

Now the execute handler:

```rust
#[entry_point]
pub fn execute(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    let mut contract_state = STATE
        .load(deps.as_ref().storage)
        .map_err(ContractError::Std)?;

    if contract_state.owner != info.sender.to_string() {
        return Err(ContractError::Unauthorized {});
    }
    match msg {
        ExecuteMsg::IncrementCounter {} => {
            contract_state.counter.checked_add(1).unwrap();
            STATE.save(deps.storage, &contract_state)?;
            Ok(Response::new()
                .add_attribute("method", "increment")
                .add_attribute("count", contract_state.counter.to_string()))
        }
        ExecuteMsg::DecrementCounter {} => {
            contract_state.counter.checked_sub(1).unwrap();
            STATE.save(deps.storage, &contract_state)?;
            Ok(Response::new()
                .add_attribute("method", "decrement")
                .add_attribute("count", contract_state.counter.to_string()))
        }
        ExecuteMsg::UpdateCounter { count } => {
            contract_state.counter = count;
            STATE.save(deps.storage, &contract_state)?;
            Ok(Response::new()
                .add_attribute("method", "update")
                .add_attribute("count", contract_state.counter.to_string()))
        }
        ExecuteMsg::ResetCounter {} => {
            contract_state.counter = 0;
            STATE.save(deps.storage, &contract_state)?;
            Ok(Response::new()
                .add_attribute("method", "reset")
                .add_attribute("count", contract_state.counter.to_string()))
        }
    }
}
```

This function enforces several layered invariants:

**The ownership invariant** — before any mutation happens, the handler checks that `info.sender` matches the stored `owner`. This is a single check that gates *every* execute path. The pattern is deliberate: ownership is checked once, at the top, before the match. If this check fails, no state is read beyond the initial load, no state is written, and the transaction rolls back. The invariant: **only the owner can mutate this contract's state**.

**Arithmetic safety** — `checked_add` and `checked_sub` return `None` on overflow/underflow instead of wrapping. The invariant: the counter value is always the result of a mathematically valid operation. An increment at `u64::MAX` or a decrement at `0` will panic rather than silently wrap to an incorrect value.

**Exhaustive matching** — the `match` on `ExecuteMsg` is exhaustive. If a new variant is added to the enum, the compiler will refuse to build until the new case is handled. The invariant: **every message type has a defined handler**. This is not enforced by tests or documentation — it is enforced by the compiler.

## Query Messages: The Read-Only Invariant

Query messages are structurally different from execute messages. They cannot mutate state. This is not a guideline — it is a type-level fact.

```rust
#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    #[returns(u64)]
    GetCount {},
}
```

The `#[returns(u64)]` annotation declares what the query returns. This is metadata for client code generation, but it also documents an invariant: **`GetCount` always returns a `u64`**. The response shape is fixed.

The query handler:

```rust
#[entry_point]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::GetCount {} => {
            let state = STATE.load(deps.storage).unwrap();
            Ok(to_binary(&state.counter).unwrap())
        }
    }
}
```

Notice what is absent: there is no `MessageInfo` parameter. Query handlers do not know who is asking. The invariant: **queries are permissionless and anonymous**. There is no access control on reads. If you need gated reads, you use a different pattern (smart query with auth tokens), but the default query model assumes public readability.

And again: `deps` here is `Deps`, not `DepsMut`. The handler physically cannot call `STATE.save(...)` because `Deps` does not provide mutable storage access. The read-only invariant is not checked at runtime. It is unrepresentable at compile time.

## The Invariant Map

Stepping back, the message and entry point design in CosmWasm maps cleanly to a set of named invariants:

| Component | Invariant |
|:--|:--|
| `InstantiateMsg` type | Initial state is always well-typed |
| `instantiate` handler | State is fully initialized or the transaction rolls back |
| `owner` field | Authorization has a single source of truth |
| `ExecuteMsg` enum | The set of possible mutations is closed and exhaustive |
| Ownership check | Only authenticated owners mutate state |
| `checked_add` / `checked_sub` | Arithmetic never silently overflows |
| `QueryMsg` / `Deps` | Queries never mutate state (compile-time guarantee) |
| `Result<T, ContractError>` | Errors are explicit; failures roll back atomically |

Each of these is a property that must always hold. Not sometimes. Not usually. Always. The combination of Rust's type system and CosmWasm's architecture makes most of them enforceable at compile time — which is the strongest guarantee a system can offer.

---

The source code is available on [GitHub](https://github.com/shreyasbhat0/cosmwasm101).
