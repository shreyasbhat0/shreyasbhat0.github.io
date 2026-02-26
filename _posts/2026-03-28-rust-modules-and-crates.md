---
title: "Modules and Crates: The Visibility Invariant"
date: 2026-03-28 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, modules, crates, visibility, use, invariants]
description: "Everything in Rust is private by default. Modules enforce encapsulation at the language level — internal implementation stays internal unless you explicitly expose it. This is how Rust makes invalid states unrepresentable from outside your API."
---

As your Rust project grows beyond a single file, you need a way to organize code, control what is visible to the outside world, and prevent internal details from leaking. Modules and crates are Rust's answer. And unlike many languages where visibility is advisory ("please don't use this private method"), Rust's visibility rules are enforced by the compiler. Break them and the code does not compile.

## Crates

A **crate** is the smallest unit of compilation in Rust. When you run `rustc` or `cargo build`, you are compiling a crate. There are two kinds:

- **Binary crate** — has a `main` function, compiles to an executable. The entry point is `src/main.rs`.
- **Library crate** — has no `main` function, compiles to a library others can use. The entry point is `src/lib.rs`.

When you run `cargo new my_project`, you get a binary crate. When you run `cargo new my_lib --lib`, you get a library crate. A package can contain both — one library crate and one or more binary crates.

Every crate you have used so far in this series has been a binary crate with a single `src/main.rs`. That is about to change.

## Packages

A **package** is a bundle of one or more crates, described by a `Cargo.toml` file. A package can contain:

- At most **one** library crate
- **Any number** of binary crates
- At least **one** crate (library or binary)

The standard layout:

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs      # binary crate root
│   └── lib.rs       # library crate root (optional)
```

If both files exist, you have a package with two crates. The library crate shares the package name. Binary crates can also live in `src/bin/`, each file becoming its own binary.

## Modules

Modules are how you organize code *within* a crate. They control two things: **structure** (where code lives) and **visibility** (who can access it).

Define a module with the `mod` keyword:

```rust
mod kitchen {
    fn wash_dishes() {
        println!("Washing dishes...");
    }

    fn chop_vegetables() {
        println!("Chopping vegetables...");
    }
}

fn main() {
    kitchen::wash_dishes(); // ERROR
}
```

```
error[E0603]: function `wash_dishes` is private
 --> src/main.rs:10:14
  |
10|     kitchen::wash_dishes();
  |              ^^^^^^^^^^^ private function
  |
note: the function `wash_dishes` is defined here
 --> src/main.rs:2:5
  |
2 |     fn wash_dishes() {
  |     ^^^^^^^^^^^^^^^^^
```

And there it is. **Everything in a module is private by default.** You defined the functions. They exist. But nothing outside the module can touch them. This is not a suggestion or a naming convention like Python's leading underscore — it is a compile-time wall.

## The `pub` Keyword

To make something accessible outside its module, mark it `pub`:

```rust
mod kitchen {
    pub fn wash_dishes() {
        println!("Washing dishes...");
    }

    fn chop_vegetables() {
        println!("Chopping vegetables...");
    }
}

fn main() {
    kitchen::wash_dishes();    // works
    // kitchen::chop_vegetables(); // still ERROR — still private
}
```

`wash_dishes` is public — anyone can call it. `chop_vegetables` remains private — only code inside `kitchen` can use it. You decide what is part of your API and what is an implementation detail. The compiler enforces the boundary.

This applies to everything: functions, structs, enums, constants, traits, and even nested modules. The default is always private.

## Module Tree and Paths

Modules form a tree. The crate root (`main.rs` or `lib.rs`) is the root of this tree. You navigate it with paths:

```rust
mod restaurant {
    pub mod front_of_house {
        pub fn seat_guests() {
            println!("Seating guests...");
        }
    }

    mod back_of_house {
        pub fn cook_order() {
            println!("Cooking...");
        }

        fn do_secret_recipe() {
            println!("You'll never know...");
        }
    }

    pub fn take_order() {
        front_of_house::seat_guests();
        back_of_house::cook_order(); // works — same parent module
    }
}

fn main() {
    restaurant::take_order();
    restaurant::front_of_house::seat_guests();

    // restaurant::back_of_house::cook_order(); // ERROR — module is private
}
```

The visibility rule: **a child module can see everything in its parent, but a parent cannot see private items in its children**. Sibling modules cannot see each other's private items either. The access flows upward, not downward or sideways.

`take_order` can call `back_of_house::cook_order()` because `take_order` lives in `restaurant`, which is the parent of `back_of_house`. But `main` cannot reach `back_of_house` directly because the module itself is not `pub`.

## Structs and Visibility

This is where the encapsulation invariant gets powerful. Consider a struct with mixed visibility:

```rust
mod auth {
    pub struct User {
        pub username: String,
        password_hash: String, // private!
    }

    impl User {
        pub fn new(username: String, password: String) -> Self {
            let password_hash = format!("hashed_{}", password); // pretend hashing
            User {
                username,
                password_hash,
            }
        }

        pub fn verify(&self, password: &str) -> bool {
            self.password_hash == format!("hashed_{}", password)
        }
    }
}

fn main() {
    let user = auth::User::new(
        String::from("alice"),
        String::from("supersecret"),
    );

    println!("Username: {}", user.username);
    // println!("Hash: {}", user.password_hash); // ERROR
    println!("Verified: {}", user.verify("supersecret"));
}
```

```
error[E0616]: field `password_hash` of struct `User` is private
 --> src/main.rs:24:39
  |
24|     println!("Hash: {}", user.password_hash);
  |                                ^^^^^^^^^^^^^ private field
```

The struct `User` is public. The field `username` is public. But `password_hash` is private. Code outside `auth` cannot read it, write it, or even acknowledge its existence in a struct literal. The only way to create a `User` is through `User::new`, and the only way to check a password is through `verify`.

This is the pattern: **private fields + public constructor = enforced invariants**. The `new` function is the gatekeeper. It can validate inputs, hash passwords, set defaults — whatever the invariant requires. Outside code cannot bypass it. You physically cannot construct a `User` with a raw password string where the hash should be, because you cannot access that field.

Combined with what we covered in the structs post, this is how you make invalid states unrepresentable. The module boundary is the enforcement mechanism.

## The `use` Keyword

Typing full paths gets verbose. The `use` keyword brings items into scope:

```rust
mod sound {
    pub mod instrument {
        pub fn guitar() {
            println!("*guitar noises*");
        }

        pub fn drums() {
            println!("*drum noises*");
        }
    }
}

use sound::instrument;

fn main() {
    instrument::guitar();
    instrument::drums();
}
```

You can also bring specific items directly into scope:

```rust
use sound::instrument::guitar;

fn main() {
    guitar(); // no prefix needed
}
```

And group multiple imports:

```rust
use sound::instrument::{guitar, drums};
```

Or bring everything in with the glob operator:

```rust
use sound::instrument::*;
```

The glob can be convenient but it makes it harder to tell where a name came from. Prefer explicit imports in production code.

## Re-exporting with `pub use`

Sometimes your internal module structure does not match the API you want to expose. `pub use` lets you re-export items from a different location:

```rust
mod internals {
    pub mod parser {
        pub fn parse(input: &str) -> Vec<String> {
            input.split(',').map(String::from).collect()
        }
    }
}

// Re-export at the crate root
pub use internals::parser::parse;

fn main() {
    let result = parse("one,two,three");
    println!("{:?}", result); // ["one", "two", "three"]
}
```

Users of your library call `parse()` directly. They never need to know about `internals::parser`. You can reorganize your internal structure without breaking the public API. The external invariant (what the API looks like) is decoupled from the internal invariant (how the code is organized).

## Separating Modules into Files

As modules grow, you move them into separate files. Rust has a specific convention for this.

Given this inline module:

```rust
// src/main.rs
mod config {
    pub fn load() -> String {
        String::from("default_config")
    }
}

fn main() {
    println!("{}", config::load());
}
```

You can split it into its own file:

```
src/
├── main.rs
└── config.rs
```

```rust
// src/config.rs
pub fn load() -> String {
    String::from("default_config")
}
```

```rust
// src/main.rs
mod config; // tells the compiler: load config from config.rs

fn main() {
    println!("{}", config::load());
}
```

The `mod config;` declaration in `main.rs` tells the compiler to look for either `src/config.rs` or `src/config/mod.rs`. The module's contents come from the file, but the visibility and position in the module tree are still determined by where `mod config;` appears.

For nested modules, use directories:

```
src/
├── main.rs
├── network/
│   ├── mod.rs
│   ├── client.rs
│   └── server.rs
```

```rust
// src/network/mod.rs
pub mod client;
pub mod server;
```

```rust
// src/main.rs
mod network;

fn main() {
    network::client::connect();
    network::server::listen();
}
```

The file structure mirrors the module tree. Each `mod` declaration is a door — and whether it has `pub` on it determines whether outside code can walk through.

## A Practical Module Layout

Here is a small project structure that puts it all together:

```
my_app/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── db/
│   │   ├── mod.rs
│   │   └── connection.rs
│   └── api/
│       ├── mod.rs
│       └── handlers.rs
```

```rust
// src/lib.rs
pub mod db;
pub mod api;
```

```rust
// src/db/mod.rs
mod connection; // private — internal detail

pub fn initialize() -> String {
    connection::connect()
}
```

```rust
// src/db/connection.rs
pub(super) fn connect() -> String {
    String::from("connected to database")
}
```

```rust
// src/main.rs
use my_app::db;
use my_app::api;

fn main() {
    println!("{}", db::initialize());
}
```

Notice `pub(super)` on `connect()`. This means "public to the parent module only." The `db` module can call `connection::connect()`, but nothing outside `db` can. Rust provides several levels of visibility:

| Syntax | Visibility |
|:--|:--|
| (default) | Private to the current module |
| `pub` | Public to everyone |
| `pub(crate)` | Public within the current crate only |
| `pub(super)` | Public to the parent module only |
| `pub(in path)` | Public to a specific ancestor module |

Each level is a different invariant about who can access the item. You pick the narrowest visibility that works.

## The Visibility Invariant, Summarized

| Mechanism | Invariant Enforced |
|:--|:--|
| Private by default | Nothing is accidentally exposed |
| `pub` on functions | Explicit API surface |
| Private struct fields | Construction only through approved paths |
| `pub use` | Public API decoupled from internal structure |
| `pub(crate)` / `pub(super)` | Fine-grained access boundaries |
| Module file structure | Code organization mirrors logical structure |

---

*Modules are where Rust's privacy rules turn into architectural tools. Private by default means your internal implementation stays internal — not by convention, not by documentation, but by compiler enforcement. Combined with private struct fields and public constructors, modules let you build APIs where the only way to create and manipulate data is through the paths you explicitly provide. Invalid states from the outside become unrepresentable.*
