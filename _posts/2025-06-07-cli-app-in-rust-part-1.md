---
title: "Building a CLI App in Rust, Part 1: Setup and the Invariants of a Well-Defined Interface"
date: 2025-06-07 10:00:00 +0530
categories: [Rust, CLI]
tags: [rust, clap, cli, invariants, cryptography]
description: "Every CLI application is a contract between the program and the user. Clap helps us encode that contract as invariants — required arguments, valid subcommands, and structured input — enforced before a single line of business logic runs."
---

A command-line interface is a contract. The user promises to provide certain arguments in a certain shape. The program promises to reject anything that violates that shape before executing any logic. This is an invariant: **the program never operates on invalid input**.

Most CLI bugs come from violating this invariant — missing arguments silently defaulting, flags being ignored, subcommands accepting nonsensical combinations. A well-designed CLI makes these violations impossible.

This is the first post in a series where we build **Cryptifer**, a file encryption CLI in Rust using [Clap](https://crates.io/crates/clap). Each post will result in a working program. By the end of the series, we will have a tool that generates cryptographic keys, encrypts files, and decrypts them — with proper error handling throughout.

I strongly recommend typing out the code rather than copying and pasting. This is one of the most effective ways to internalize what each piece does.

## Overview

Cryptifer will have three commands, each with its own invariants:

- **Generate**: Produces a random key and IV (initial value) and writes them to an output file. Invariant: *the output path must be specified*.
- **Encrypt**: Encrypts a given file using a key and IV. Invariant: *both a file path and a key path must be provided*.
- **Decrypt**: Decrypts an encrypted file using a key and IV. Invariant: *both an encrypted file path and a key path must be provided*.

Clap will enforce all of these at the argument-parsing layer — before `main()` runs any application logic.

## Getting Started

### Installation

If you do not have Rust installed:

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Creating a New Binary Package

```
cargo new cryptifer --bin
```

This creates a new Rust project with a `src/main.rs` entry point.

### Adding Dependencies

```
cargo add clap --features derive
```

The `derive` feature gives us procedural macros that generate argument-parsing code from struct and enum definitions. This is where Clap's power lies: we define the *shape* of valid input as Rust types, and Clap generates the parser that enforces that shape.

## Setting Up Commands

Create a new file called `commands.rs` in the `src` directory and add the following imports to `main.rs`:

```rust
use clap::{Parser, Subcommand};
mod commands;
use commands::*;
```

Now define the command structure in `commands.rs`. The struct `Cryptifer` is the root command, and `Commands` is an enum of subcommands:

```rust
use super::*;

#[derive(Debug, Parser)]
#[clap(
    name = "Cryptifer",
    about = "Cryptifer is a CLI Application to Encrypt and Decrypt the file",
    version = "0.0.1"
)]
pub struct Cryptifer {
    #[clap(subcommand)]
    pub command: Commands,
}

#[derive(Debug, Subcommand)]
pub enum Commands {
    /// Generates Keystore to out file given
    #[clap(arg_required_else_help = true)]
    Generate {
        #[clap(short = 'o', long)]
        output_path: String,
    },
    /// Encrypts file specified using keypath
    #[clap(arg_required_else_help = true)]
    Encrypt {
        #[clap(short = 'f', long)]
        file_path: String,
        #[clap(short = 'k', long)]
        key_path: String,
    },
    /// Decrypts file specified using keypath
    #[clap(arg_required_else_help = true)]
    Decrypt {
        #[clap(short = 'f', long)]
        encrypted_file: String,
        #[clap(short = 'k', long)]
        key_path: String,
    },
}
```

There are several invariants encoded in this definition:

1. **A subcommand is mandatory.** `Cryptifer` has a `command` field of type `Commands` — it is not `Option<Commands>`. The parser will reject invocations with no subcommand.
2. **Each subcommand's arguments are mandatory.** `arg_required_else_help = true` means that if the user runs `cryptifer generate` without `-o`, Clap prints help text and exits rather than proceeding with a missing path.
3. **The argument types are enforced.** Every flag is a `String` — Clap handles the conversion from raw CLI input and rejects anything it cannot parse.

These are compile-time decisions about the shape of valid input. Clap turns them into runtime enforcement. The key insight: **the best input validation is the kind that runs before your code does**.

### Configuring `main.rs`

```rust
use clap::{Parser, Subcommand};
mod commands;
use commands::Cryptifer;

fn main() {
    Cryptifer::parse();
}
```

At this point, `parse()` consumes the CLI arguments, validates them against the struct definition, and either returns a valid `Cryptifer` instance or exits with an error message. No manual argument checking. No `if args.len() < 2`. The invariant — "all required arguments are present and well-formed" — is upheld by the parser.

### Running the CLI

```
cargo run -- --help
```

You should see the help output with all three subcommands listed, their descriptions, and their required flags. The program compiles and runs. It does not do anything yet — but it already enforces the contract of what valid input looks like.

## Wrapping Up

This is the foundation. We have a CLI skeleton that enforces structural invariants on every invocation: a subcommand must be provided, its required flags must be present, and their types must be valid. No business logic can execute with malformed input.

In the next post, we will implement the `generate` command — producing cryptographic keys and initial values. That introduces a new invariant: the generated secret must have exactly the right byte length for the cipher we will use later.
