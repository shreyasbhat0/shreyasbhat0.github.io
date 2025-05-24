---
title: "Rust Fundamentals: Variables, Mutability, and the Type Invariant"
date: 2025-03-22 10:00:00 +0530
categories: [Rust, Fundamentals]
tags: [rust, variables, mutability, data-types, invariants]
description: "Rust is statically typed — every value has a type known at compile time. This isn't a convenience feature. It's an invariant that eliminates entire categories of runtime failures."
---

Rust is a statically typed language. Every value has a type, and every type is known at compile time. This is not a stylistic choice — it's an invariant the compiler enforces. The consequence: if your program compiles, every operation on every value is type-safe. No "undefined is not a function." No "cannot read property of null." These errors are structurally impossible.

## Variables and Mutability

### Variables

Variables in Rust are declared with the `let` keyword, followed by a name and optionally a type annotation:

```rust
fn main() {
    let x: i32; // x is declared as type i32
    x = 10;
    println!("The value of x is: {}", x);
}
```

The compiler can also infer the type from the assigned value:

```rust
fn main() {
    let x = 10; // inferred as i32
    println!("The value of x is: {}", x);
}
```

Now here's where Rust's invariant thinking shows up immediately. What happens when you try to reassign `x`?

```rust
fn main() {
    let x: i32;
    x = 10;
    x = 20; // ERROR
    println!("The value of x is: {}", x);
}
```

The compiler rejects this:

```
error[E0384]: cannot assign twice to immutable variable `x`
```

### The Immutability Invariant

Variables in Rust are **immutable by default**. This is a deliberate design decision that establishes an invariant: *once a value is bound to a name, that binding does not change*. Code that reads `x` later can trust that the value hasn't been silently mutated somewhere in between.

To opt into mutability, you use `mut` explicitly:

```rust
fn main() {
    let mut x: i32;
    x = 10;
    x = 20;
    println!("The value of x is: {}", x); // prints 20
}
```

The `mut` keyword is a signal — to the compiler, to other developers, and to your future self — that this value *will* change. Mutability is not the default; it's a conscious decision that you mark in the code. This is the opposite of most languages, where everything is mutable unless you go out of your way to freeze it.

### Constants

Constants are values bound to a name that can never change. They are declared with `const` and require a type annotation:

```rust
const APPLICATION_NAME: &str = "First Constant";
```

Constants differ from immutable variables in important ways:

- They cannot use `mut` — ever.
- They must have a value known at compile time.
- They are valid for the entire duration of the program, within their declared scope.
- By convention, they use `SCREAMING_SNAKE_CASE`.

Constants are invariants made explicit in your code. `APPLICATION_NAME` will be `"First Constant"` at every point in the program's execution. The compiler guarantees it.

## Data Types

Every value in Rust has a type known at compile time. The compiler infers types where possible, but the invariant is absolute: **ambiguity is not allowed**. If the compiler cannot determine the type, it will not compile the program.

### Integers

Integers are numbers without fractional components. Rust provides both signed and unsigned variants:

- **Signed**: `i8`, `i16`, `i32`, `i64`, `i128` — can store values from -(2^(n-1)) to 2^(n-1) - 1
- **Unsigned**: `u8`, `u16`, `u32`, `u64`, `u128` — can store values from 0 to 2^n - 1

The type invariant for integers: a value of type `u8` is always in the range `[0, 255]`. Always. Overflow in debug mode panics. Overflow in release mode wraps (or you can use `checked_*`, `wrapping_*`, `saturating_*` methods to be explicit about the behavior you want). Either way, you cannot silently corrupt data.

### Floating Point Numbers

Rust has two floating-point types: `f32` (single precision) and `f64` (double precision). The default is `f64`.

```rust
fn main() {
    let x = 10.0;       // f64 by default
    let y: f32 = 120.0; // explicitly f32
}
```

All floating-point numbers are signed.

### Character Type

Rust's `char` type is four bytes and represents a Unicode Scalar Value — not just ASCII.

```rust
fn main() {
    let ch = 'a';
    let z: char = 'Z';
    let surprised_cat = '🙀';
}
```

The invariant: every `char` is a valid Unicode Scalar Value. You cannot construct an invalid character.

### Booleans

One byte, two possible values: `true` and `false`.

```rust
fn main() {
    let x = true;
    let y: bool = false;
}
```

Unlike C, Rust does not implicitly convert integers to booleans. `if 1` is a type error. The invariant is strict: conditions must be boolean expressions, period.

### Tuples

Tuples are fixed-size, heterogeneous, ordered sequences:

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
    println!("The value of tuple is: {:?}", tup);
}
```

Destructuring extracts individual elements:

```rust
fn main() {
    let tup = (500, 6.4, 1);
    let (x, y, z) = tup;
    println!("The value of z is: {:?}", z);
}
```

The tuple invariant: the type and count of elements is fixed at compile time. A `(i32, f64, u8)` always has exactly three elements of exactly those types. You cannot index out of bounds — the compiler rejects it.

### Arrays

Arrays are fixed-length, homogeneous collections allocated on the stack:

```rust
fn main() {
    let a: [i32; 5] = [1, 2, 3, 4, 5]; // 5 elements of type i32
    let b = [3; 5];                       // 5 elements, all with value 3
    let last = a[4];                      // accessing the last element
}
```

Iterating over an array:

```rust
fn main() {
    let a: [i32; 5] = [1, 2, 3, 4, 5];
    for index in 0..a.len() {
        println!("{}", a[index]);
    }
}
```

The critical invariant: **array bounds are checked at runtime**. Access `a[10]` on a 5-element array and the program panics — it does not silently read garbage memory. This single invariant eliminates the buffer overflows that have caused decades of security vulnerabilities in C and C++.

## The Type Invariant, Summarized

Every primitive type in Rust carries a guarantee:

| Type | Invariant |
|:--|:--|
| `i32` | Always a valid 32-bit signed integer |
| `bool` | Always `true` or `false`, never `0` or `1` |
| `char` | Always a valid Unicode Scalar Value |
| `[T; N]` | Always exactly `N` elements, always bounds-checked |
| `(T, U)` | Always exactly two elements of types `T` and `U` |

These aren't runtime checks you have to remember. They're compile-time guarantees that hold for every value, in every function, in every module of your program.

---

*Variables in Rust aren't just storage locations — they're contracts. Immutable by default, typed by requirement, bounds-checked by guarantee. The next post covers control flow and how Rust enforces type consistency across branches.*
