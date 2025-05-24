---
title: "Cargo: Rust's Package Manager and the Invariants of Reproducible Builds"
date: 2025-03-08 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, cargo, package-manager, invariants]
description: "Cargo isn't just a convenience tool. It enforces a set of invariants about how Rust projects are structured, built, and distributed — so you can't accidentally break the build."
---

A package manager is more than a download tool. It's a system that maintains invariants about your project's dependencies, build process, and artifact structure. Get any of these wrong and the build fails, the deployment breaks, or — worse — it works on your machine and nowhere else.

Cargo is Rust's package manager, and it enforces these invariants from the start.

## What Cargo Does

Cargo handles four things:

1. **Metadata** — introduces two metadata files (`Cargo.toml` and `Cargo.lock`) with package information.
2. **Dependencies** — fetches and builds your package's dependencies.
3. **Compilation** — invokes `rustc` with the correct parameters to build your package.
4. **Conventions** — enforces a standard project structure so every Rust project looks the same.

That last point is an invariant most people overlook. When every Rust project follows the same directory layout, tools can make assumptions. CI pipelines work without project-specific configuration. New contributors navigate immediately. The convention *is* the invariant.

> Cargo is installed alongside Rust via `rustup`. Alternatively, you can build it from [source](https://github.com/rust-lang/cargo#compiling-from-source).

## Creating a New Package

```bash
mkdir projects
cd projects
cargo new hello_cargo --bin
```

`cargo new <name> --bin` creates a new binary program. Use `--lib` instead for a library package.

The generated directory structure:

```
hello_cargo/
├── Cargo.toml
└── src/
    └── main.rs
```

This structure is not a suggestion — it's a contract. Cargo expects `src/main.rs` for binaries and `src/lib.rs` for libraries. Violate this convention and nothing builds.

## Cargo.toml: The Manifest

`Cargo.toml` is written in [TOML](https://toml.io/en/) and contains all the metadata Cargo needs to compile your package: name, version, edition, and dependencies.

The invariant here is **declarative completeness** — everything required to build the project is declared in this file. No hidden environment variables. No "make sure you install X first" tribal knowledge. If it's not in `Cargo.toml`, it's not a dependency.

## Adding Dependencies

Use `cargo add` to add a dependency:

```bash
cargo add rand
```

This updates `Cargo.toml` under the `[dependencies]` section:

```toml
[dependencies]
rand = "0.8"
```

You can also edit `Cargo.toml` directly. Either way, the invariant holds: **the manifest is the single source of truth for dependencies**.

## Building and Running

Build the project:

```bash
cargo build
```

This compiles your code and all dependencies, placing the binary in `target/debug/`.

Run the project:

```bash
cargo run
```

This builds (if necessary) and executes in one step. Cargo checks whether source files have changed and only recompiles what's needed — maintaining the invariant that **the binary always reflects the current source**.

## The Lock File Invariant

When you first build, Cargo generates `Cargo.lock`. This file pins the exact versions of every dependency that was resolved. The invariant: **given the same `Cargo.lock`, the same versions are always used**. This is what makes builds reproducible.

For binary projects, you commit `Cargo.lock`. For libraries, you typically don't — because the library's consumers should resolve their own dependency versions.

---

*Cargo seems simple on the surface, but it encodes a set of invariants about reproducibility, structure, and dependency management that eliminate entire classes of "works on my machine" failures. The next post covers the specific commands Cargo provides.*
