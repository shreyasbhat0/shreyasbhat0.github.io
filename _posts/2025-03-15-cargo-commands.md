---
title: "Cargo Commands: The Interface to Rust's Build Invariants"
date: 2025-03-15 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, cargo, build, testing, publishing, invariants]
description: "Every Cargo command upholds a specific contract. Build guarantees compilation. Test guarantees verification. Publish guarantees distribution. Here's the full reference."
---

Every Cargo command is an operation that transforms your project from one valid state to another. `cargo build` takes source and produces binaries. `cargo test` takes code and produces a pass/fail verdict. `cargo publish` takes a local package and produces a distributed crate. Each command upholds specific invariants about what it accepts and what it produces.

## Build Commands

### cargo build

Compiles the current package.

```bash
cargo build [OPTIONS]
```

**Package Selection:**

When no package is specified, Cargo builds based on the manifest in the current working directory.

- `--package <spec>` — build only the specified packages
- `--workspace` — build all members in the workspace
- `--exclude <spec>` — exclude specified packages from the build

**Target Selection:**

By default, Cargo builds all binary targets.

- `--lib` — build the package's library
- `--bin <name>` — build the specified binary
- `--bins` — build all binary targets
- `--example <name>` — build the specified example

**Feature Selection:**

Features are compile-time flags that enable or disable sections of code. The invariant: **the same feature set always produces the same compilation result**.

- `--features <features>` — space or comma-separated list of features to activate
- `--all-features` — activate all available features
- `--no-default-features` — disable the default feature set

**Output Selection:**

- `--target-dir <dir>` — directory for all generated artifacts
- `--out-dir <dir>` — copy final artifacts to this directory (nightly only, requires `-Z unstable-options`)

### cargo run

Builds and runs the current package.

```bash
cargo run [OPTIONS] [-- args]
```

Arguments after `--` are passed to the binary, not to Cargo. This separation is itself an invariant — Cargo's arguments and the binary's arguments never mix.

### cargo test

Executes unit, integration, and documentation tests.

```bash
cargo test [OPTIONS] [testname] [-- test-options]
```

The invariant `cargo test` upholds: **if it returns exit code 0, every test passed**. No partial success, no "mostly passing." Zero or non-zero. Binary.

Arguments after `--` go to the test harness (libtest). For details, see `cargo test -- --help`.

### cargo doc

Builds documentation for the package and its dependencies.

```bash
cargo doc [OPTIONS]
```

- `--open` — open the docs in a browser after building
- `--no-deps` — skip documentation for dependencies
- `--document-private-items` — include non-public items

The invariant here is **documentation completeness** — every public item gets documented, and `cargo doc` fails if doc comments contain broken links or invalid code examples (with `#[deny(rustdoc::broken_intra_doc_links)]`).

## Package Commands

### cargo init

Creates a new Cargo package in an existing directory.

```bash
cargo init [OPTIONS] [path]
```

### cargo new

Creates a new Cargo package in a new directory.

```bash
cargo new [OPTIONS] <path>
```

This generates the standard project structure: `Cargo.toml`, `src/`, and a VCS ignore file. The invariant is **structural correctness from the start** — the generated project compiles immediately.

### cargo install

Builds and installs a Rust binary.

```bash
cargo install [OPTIONS] crate[@version]
cargo install [OPTIONS] --path <path>
cargo install [OPTIONS] --git <url>
cargo install [OPTIONS] --list
```

Only packages with `[[bin]]` or `[[example]]` targets can be installed. The installation root is resolved in order of precedence:

1. `--root` option
2. `CARGO_INSTALL_ROOT` environment variable
3. `install.root` Cargo config value
4. `CARGO_HOME` environment variable
5. `$HOME/.cargo`

## Publishing Commands

### cargo login

Authenticates with crates.io.

```bash
cargo login [OPTIONS] [token]
```

The API token is saved to `$CARGO_HOME/credentials.toml`. If no token argument is provided, it reads from stdin. Keep this token secret — it grants publish access to your crates.

### cargo publish

Uploads a package to the registry.

```bash
cargo publish [OPTIONS]
```

This performs several steps:

1. Checks `package.publish` in the manifest for registry restrictions
2. Creates a compressed `.crate` file (following `cargo package` steps)
3. Uploads the crate to the registry (default: crates.io)
4. The server performs additional validation

The invariant `cargo publish` enforces: **a published crate is immutable**. Once version `1.2.3` is published, that exact version can never be replaced with different code. You can yank it (mark it as discouraged), but you cannot change it. This is the foundation of dependency trust in the Rust ecosystem.

For more details on packaging and publishing, see the [official documentation](https://doc.rust-lang.org/cargo/reference/publishing.html).

---

*These commands form the interface through which you interact with Cargo's invariants — reproducible builds, structural conventions, and immutable publications. The full reference is at [doc.rust-lang.org/cargo](https://doc.rust-lang.org/cargo).*
